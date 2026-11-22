<!-- Trusted provenance: F-f0a9e1163cfa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f0a9e1163cfa

## Purpose

F-f0a9e1163cfa performs an ordered chain of indirect interactions, transforms incoming and returned machine words, and performs extensive mixed-width memory updates before an unresolved final transfer.

No application-level purpose is established. Character-like constants, repeated offsets, and pointer-like words do not establish text formatting, object management, device initialization, or a protocol.

This contract distinguishes:

- **Established static evidence:** interactions, arithmetic, access widths, and ordering visible in the supplied representation.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** behavior not established by the available evidence.

The requirements are conditional on the supplied representation accurately describing an executable path. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has four inputs and no declared return value.

| Input | Width | Established use |
|---|---:|---|
| First | 32 bits | Forwarded to the first indirect interaction. |
| Second | 32 bits | Forwarded to the first indirect interaction. |
| Third | 32 bits | No explicit use in the visible body. |
| Fourth | Signed 32-bit representation | Shifted left seven positions and supplied twice to an early interaction. |

The first three inputs have no established signedness, encoding, pointer interpretation, or ownership meaning. Absence of an explicit use of the third input does not prove irrelevance under the unknown complete calling convention.

### Additional execution inputs

Execution also depends on:

- Two incoming 32-bit data-register values outside the explicit interface.
- An incoming control-link value used to derive the first target.
- Secondary result-register contents following several interactions.
- Fixed-location and computed-address memory contents.
- A final indirect target whose provenance is unresolved.

The additional incoming data words participate in logical right shifts by eleven or twelve positions. Their initial values are unknown; zero initialization is not established.

### Outputs and completion

There is no established scalar result, success status, output length, or ownership token.

Observable outputs comprise ordered dependency interactions, memory accesses, and a final four-argument transfer. A return is represented after that transfer, but actual normal return, nonreturning dispatch, dependency termination, and faults remain **documented_only**.

### Preconditions

No visible validation establishes safe arguments, valid targets, writable storage, or alignment.

To complete without an access or control-flow failure, every encountered target and memory access must be valid at its represented width. This includes low fixed addresses and destinations derived from returned words, loaded words, zero-extended bytes, and arithmetic.

No allocation guarantee, null exemption, bounds check, or alignment repair is established.

## Behavior

### Widths and arithmetic

Bytes are eight bits, halfwords sixteen bits, and words thirty-two bits. Several interaction results are represented as sixty-four bits and split into low and high thirty-two-bit words. This establishes value decomposition, not memory byte order.

Byte stores retain the low eight bits. Unsigned byte reads contribute values from 0 through 255 when promoted to words or used as address bases.

Established unsigned transformations include:

| Transformation | Result property |
|---|---|
| Low eleven bits shifted left ten | At most 0x1FFC00 |
| Bits 8–23 extracted | At most 0xFFFF |
| Bits 12–23 extracted | At most 0xFFF |
| Bits 19–23 extracted | At most 31 |
| Bits 11–17 extracted | At most 127 |
| Bits 4–16 extracted | At most 0x1FFF |
| Word shifted left eight, masked to fifteen bits, then shifted right six | Multiple of four from 0 through 508 |

Other expressions have signed thirty-two-bit representations, including multiplication, left shifts, address calculations, and a variable right shift. Overflow, address wrapping, and oversized shift behavior require target semantics. Where this contract uses ordinary truncating thirty-two-bit arithmetic, it states that condition rather than imposing it universally.

### Indirect-interaction phase

The first interaction receives the first two explicit inputs and the first additional incoming word logically shifted right twelve positions. Two interactions without represented explicit arguments follow. The next argument-bearing interaction receives two copies of the fourth input shifted left seven positions. Another pair follows, then an interaction receiving three zero words.

The subsequent ordered chain forwards returned words, secondary result-register contents, explicit zeros, and extracted fields. Recurrent argument relationships include:

- A sixty-four-bit result’s low word shifted left eight, with its high word supplied separately.
- Incoming data words logically shifted right eleven.
- A returned word’s low eleven bits shifted left ten.
- Bits 4–23 of a returned word shifted left thirteen.
- A five-bit field later shifted left three and thirty-one.
- A retained high word’s bits 11–17.
- Thirteen-bit and twenty-four-bit masked fields.
- Duplicated shifted values and explicit zero arguments.

Some retained intermediates are explicitly reset to zero. Arguments derived exclusively from those zero values remain zero under ordinary shift semantics. This does not establish preservation of other registers, globals, or memory by dependencies.

Each represented interaction retains its position in the chain. Similar arguments do not establish identical targets or interchangeable behavior. An interaction without displayed arguments has no **represented explicit arguments**; its additional ABI inputs remain unknown.

### Entry into direct memory mutation

The following descriptive labels identify captured values, not types or ownership:

- **Packed base:** a returned low word shifted left twenty-five.
- **Scaled base:** a later returned word multiplied by 0x02000000.
- **Small base:** twice the bounded, multiple-of-four field described above.
- **Payload:** a retained returned word shifted left eight.
- **Final pair:** the last represented sixty-four-bit result before direct stores.

Under truncating thirty-two-bit arithmetic, the packed and scaled bases have their low twenty-five bits clear. Their low bytes are therefore zero. The small base ranges from 0 through 1016 in multiples of eight.

The opening effects have three ordered parts.

First, the packed base is stored as a word at the final high word plus 0x14. Its plus-9 byte receives the final high word’s low byte. A word destination formed from 256 times the final low word, plus 0x54, receives the scaled base. The packed-base plus-9 byte is then overwritten with the packed base’s low byte.

Second, the scaled-base plus-0x14 word is zeroed, and a byte is read from scaled base plus 5. A zero word is written at small base plus 0x14. The captured byte is then zero-extended and stored as a word at scaled base plus 0x34.

Third, the scaled-base plus-0x14 word is read. Another zero word is written at small base plus 0x14, followed by two successive byte stores of the captured word’s low byte at small base plus 0x15.

**Stable ordinary-memory consequence:** under ordinary thirty-two-bit byte addressing, successful accesses, and no external or read-induced mutation, the captured plus-0x14 word is zero. The same-base plus-0x34 word starts 32 bytes away and cannot overlap it, including across address wraparound. The intervening small-base word store supplies only zero bytes, so overlap with that store cannot introduce a nonzero value.

A different reread requires additional behavior, such as external mutation, read side effects, or a nonordinary memory mapping. None is established. General aliasing uncertainty does not invalidate this arithmetic consequence.

### Linked and overlapping updates

The next region reads the byte at fixed address 5 and uses its zero-extended value as an address base. It also writes fixed addresses 1, 0x15, and 0x54.

Computed accesses use byte offsets including 5, 9, 0x0D, 0x11, and 0x15, and word offsets including 4, 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74. Decimal 100 and hexadecimal 0x64 denote the same displacement.

The region preserves these observable characteristics:

- A previously loaded word is captured before its source location is overwritten.
- Words loaded through plus-0x74 and plus-0x64 accesses determine subsequent destinations.
- A byte loaded through plus-5 can determine the base of a plus-0x74 word read.
- Self-referential stores place a loaded word at an address derived from that word.
- Duplicate byte and word writes occur and remain separate accesses.
- Fresh byte reads are interleaved with computed stores.

These patterns establish no structure definition, object size, or pointer ownership. Distinct-looking expressions may alias; repeated reads are not generally replaceable by earlier captures.

For later descriptions, **root word** denotes the captured word obtained through the low-address-derived plus-0x54 access. **Linked word** denotes the later plus-0x74 result reached through a freshly read byte base.

### Constants and repeated publication

The linked word determines three stores of 0x3A: a byte at four times that word, a halfword at eight times that word, and a word at four times that word plus 0x20.

An adjusted quantity is formed by subtracting 0x96 from four times the root word. **Adjusted base** denotes four times that adjusted quantity.

At adjusted base plus 0x20, four successive word stores supply:

1. 0x20.
2. 0x2F.
3. A zero-extended byte freshly read from adjusted base plus 4.
4. 0x20 again.

The intervening byte read and transient values are observable trace elements.

An **anchor** word is then read from an additional fixed location. That read precedes the word store of 0x20 at fixed address 0x60.

Publication uses four-access groups: a derived word goes to a scaled destination plus 0x54; the adjusted base’s low byte goes to anchor plus 0x19; the anchor’s low byte goes to adjusted base plus 0x11; and a scaled word goes to anchor plus 0x34.

The initial group uses eight times the root word as its scaled destination. Subsequent groups use eight times the adjusted quantity. Their published payload is four times the linked word. Repetition is established; an independently audited numeric count remains **documented_only** here. One group or a final-memory-only check is not an adequate substitute.

### Final mutation and transfer

For this region:

- **Publication payload** is four times the linked word.
- **Linked base** is twice the linked word.
- **Fixed capture** is the word read at fixed address 0xA9.
- **Publication capture** is the word read at publication payload plus 0x74.
- **Earlier linkage capture** is the word read at linked base plus 0x14.
- **Adjusted-field capture** is the word read at adjusted base plus 4.

These names distinguish retained values from later reads.

#### Opening read/write boundary

After publication, the required order is: read fixed address 0xA9 as a word; read publication payload plus 0x74 as a word; write the derived byte at fixed address 0x3A; then read linked base plus 0x14 and adjusted base plus 4 as words, in that order.

The derived byte is the low eight bits of eight times the adjusted quantity plus 0x30 under truncating arithmetic. The fixed-address write must not move after the latter reads: it may alias their storage.

The following stores use retained publication bytes. Linked base plus 0x0D receives the adjusted base’s low byte. Adjusted base plus 8 receives 0x6C, and its plus-0x0D and plus-9 bytes receive the anchor’s low byte. A word at linked base plus 0x54 receives the linked base.

#### Fixed address 0x67 and fresh consumption

Fixed address 0x67 receives the fixed capture’s low byte. A word is then read from linked base plus 0x44. Only afterward is fixed address 0x67 freshly read as a byte for the write to fixed address 0x21.

The newly loaded linked-base word next supplies a byte at adjusted base plus 0x10, a word at anchor plus 0x54, and the signed-shift-derived byte at linked base plus 1. The shift count is the adjusted base’s low eight bits. Anchor plus 0x54 is subsequently overwritten with the adjusted base.

A fresh word read at adjusted base plus 0x40 precedes word writes of the adjusted-field capture to anchor plus 0x74, the newly read word to adjusted base plus 0x10, and the linked base to anchor plus 0x44.

Fixed address 0x67 is then freshly read again for a byte write at adjusted base plus 9. This is not a reuse of the original fixed capture.

Before the next consumption of 0x67, the function writes the adjusted base’s low byte at linked base plus 9, writes the fixed capture as a word at adjusted base plus 0x44, and reads a word at adjusted base plus 0x54. Fixed address 0x67 is then freshly read a third time, again supplying a byte store at adjusted base plus 9.

Computed writes between these reads may alias 0x67. Consequently, the three consumed bytes are not unconditionally equal.

#### Remaining linkage effects

The newly loaded plus-0x54 word becomes a destination base. Its plus-0x0D byte receives the publication capture’s low byte. Linked base plus 0x54 receives the earlier linkage capture.

A fresh word read at adjusted base plus 0x14 supplies a later low-byte store. Before that store, the publication capture’s low byte is written at linked base plus 0x11, and the newly loaded destination base is stored at adjusted base plus 9 as a byte and plus 0x44 as a word.

A fresh word at linked base plus 4 determines both the halfword destination receiving 0x65 at twice that word and subsequent plus-0x64 and plus-0x74 word reads. The plus-0x64 read occurs before the captured adjusted-base field’s low byte is written at the destination base plus 0x11 and word 100 is written at its plus-0x54 location. The plus-0x74 read follows those writes.

Finally, a word is read from linked base plus 0x64. The loaded plus-4 word’s low byte is written at linked base plus 9. The last plus-0x64 result receives the earlier plus-0x64 result at its plus-0x54 destination, and is itself stored at twice that earlier result.

The final unresolved transfer receives 0x72, the earlier plus-0x64 result, the earlier linkage capture, and the later plus-0x74 result. Their application meanings are unknown.

## Side effects

### Memory and ordering

Effects are not confined to caller-provided buffers. Fixed low addresses and arbitrary computed destinations are involved.

Access widths, duplicate writes, fresh reads, and interleaving are part of the represented behavior. Captured words do not change merely because their original locations are later overwritten; fresh reads may observe those overwrites.

Memory byte order, atomicity, volatile classification, barriers, interrupt exclusion, and synchronization are not established. Ordinary-address nonoverlap conclusions must be distinguished from unknown hardware mappings or external mutation.

### Ownership and lifetime

No explicit allocation or release is identifiable. Dependencies may nevertheless allocate, release, retain arguments, invalidate storage, or modify globals.

Neither forwarding a word nor storing a pointer-like value establishes ownership transfer. No dependency is assumed to preserve globals or memory.

### Errors and partial effects

No explicit error branch, failure status, rollback, or cleanup handler is represented. A failed interaction or access may leave all earlier effects in place. No transactional guarantee is established.

## Dependencies

Only F-f0a9e1163cfa has a verified opaque identity in the supplied mapping.

| Opaque reference | Established role | Unresolved behavior |
|---|---|---|
| F-U001 | First interaction; receives the first two explicit words and an incoming word shifted right twelve | Identity, complete ABI, results, side effects, ownership |
| F-U003 onward | Document-local references for intervening interactions in occurrence order | Target equivalence, preservation, failure and memory behavior |
| F-U002 | Final transfer with four represented arguments | Identity, dispatch meaning, continuation, ownership, return |

These document-local references do not establish verified target identities or imply that different occurrences necessarily target different functions. No dependency is assumed pure, idempotent, allocation-free, or nonthrowing.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Early input transformation.** If the fourth input is 1 and preceding interactions return, the relevant interaction receives 128 twice.

2. **Masked extraction.** For a relevant returned word of 0xAB123456, extracting bits 8–23 produces 0x1234. The upper byte does not contribute.

3. **Small-base boundary.** Under truncating thirty-two-bit arithmetic, low seven bits of 0x7F produce field value 508 and small base 1016. Accessibility of small base plus 0x14 is a separate precondition.

4. **Packed-base overwrite.** Under the same arithmetic, the packed base’s low byte is zero. The second opening byte store at its plus-9 location therefore supplies zero.

5. **Opening zero reread.** With stable ordinary byte-addressed memory and successful accesses, the opening scaled-base plus-0x14 reread yields zero. The plus-0x34 word cannot overlap it even through wrapping, and the intervening potentially overlapping word store supplies zero.

6. **Post-publication ordering.** An acceptable trace places the fixed 0x3A byte write after the publication-payload plus-0x74 read but before the linked-base plus-0x14 and adjusted-base plus-4 reads.

7. **Fresh 0x67 values.** If the first computed plus-9 byte write reaches 0x67, and an intervening word store changes that byte before the next fixed-address read, the next plus-9 byte write uses the freshly read value—not the original fixed capture. Byte position within an overlapping word depends on memory byte order.

8. **Transient publication values.** The adjusted-base plus-0x20 trace includes 0x20, 0x2F, the freshly read byte promoted to a word, and 0x20 again. Final memory alone cannot demonstrate this behavior.

9. **Oversized shift.** If the masked variable shift count exceeds the signed word width, its result remains **documented_only** pending target-semantic evidence.

10. **Nonreturning dependency.** If an early dependency does not return, later accesses need not occur. Cleanup is not established.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid callers, and complete ABI.
- Indirect identities and dependency contracts.
- Signed overflow, address wrapping, and oversized shifts.
- Byte order, alignment, hardware mapping, and read side effects.
- External mutation and concurrency behavior.
- Object layouts, ownership, allocation, and release.
- An independently audited publication repetition count.
- Runtime reachability of the entire represented path.
- Final-transfer continuation and normal return.

Packed-field manipulation and linked-state publication are plausible interpretations only. They are not established intent.

## Evidence

The sole behavioral evidence is the supplied static representation associated with F-f0a9e1163cfa. Prior prose and reviewer feedback are not independent behavioral authority.

Static inspection supports the represented widths, transformations, interaction sequence, mixed-width accesses, fixed-address effects, and final argument tuple. In particular, it establishes the fixed 0x3A write boundary and three separate reads of fixed address 0x67.

The opening zero-reread conclusion is conditional arithmetic and ordinary-memory reasoning: same-base offsets 0x14 and 0x34 remain 32 bytes apart modulo the thirty-two-bit address space, while the other intervening word store contains zero.

No runtime execution, memory trace, dependency inspection, or test result supports this document. Acceptance examples express conditional consequences of static evidence, not observed execution.
