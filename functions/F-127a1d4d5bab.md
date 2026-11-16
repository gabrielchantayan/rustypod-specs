<!-- Trusted provenance: F-127a1d4d5bab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-127a1d4d5bab

## Purpose

F-127a1d4d5bab performs a sequential series of indirect invocations, transforms acquired results, mutates shared and indirectly addressed memory, and reaches an unresolved indirect invocation with four visible arguments.

**Established static evidence:** the supplied representation exposes no explicit parameters or meaningful return value. Its observable behavior includes result-dependent invocation arguments, fixed-width arithmetic, byte and word reads, byte/halfword/word stores, repeated publications, and a final indirect transfer.

**Inferred intent — [INFERENCE]:** masks, shifts, scaling, and recurring displacements suggest packed-state and link-like manipulation. They do not establish an application operation, object model, allocation protocol, or validated data format.

This contract describes static evidence. Dependency behavior and execution-model questions remain **documented_only** where unresolved. No runtime tests have been performed.

## Interface

### Inputs and outputs

The apparent interface has no explicit arguments. Effective inputs nevertheless include:

- Incoming execution context used to select the first indirect destination.
- Primary and secondary result channels of indirect invocations.
- Existing shared-memory contents, including absolute low-address storage.
- A distinct global word acquired immediately before the extended repeated-write region.
- Words and unsigned bytes acquired through calculated addresses.
- The destination and calling convention of the terminal invocation.

No ordinary return value is established. Outputs comprise the invocation sequence, visible argument values, memory acquisitions and publications, and terminal transfer.

A return follows the terminal invocation in the supplied representation. Whether that target returns normally, transfers control elsewhere, faults, or allows return to the original caller is **documented_only**.

### Widths and layouts

| Quantity | Evidenced treatment |
|---|---|
| Byte acquisition or publication | 8 bits |
| Halfword publication | 16 bits |
| General words and address-bearing values | 32 bits |
| Combined invocation result | 64 bits, consumed through low and high 32-bit portions |
| Byte-derived address component | Unsigned widening from 8 bits |
| Explicitly unsigned right shift | Logical, zero-filling |

Byte and halfword publications retain only the low eight or sixteen bits. They do not replace an entire surrounding word.

High/low result extraction does not establish memory byte order. Displacements are byte-address displacements, not validated field definitions. In particular, a word at offset 0x14 overlaps a byte at offset 0x15. Decimal 100 and hexadecimal 0x64 identify the same displacement.

### Preconditions

Progress requires executable indirect destinations, compatible invocation conventions, usable result channels, and accessible memory for every evaluated access. Any alignment requirements belong to the unresolved execution environment.

No visible validation establishes these conditions. Conventional non-null object-pointer assumptions are insufficient: the routine accesses absolute low addresses and uses unsigned bytes and scaled integers as address components.

## Behavior

### Invocation prefix and arithmetic

The invocation-heavy prefix is sequential. Most destinations are formed from execution-position-related values with displacement 0xffe; the first additionally depends on incoming context. Target identities are unresolved.

Visible argument relationships include:

- Primary results shifted left by eight.
- Low portions of combined results shifted left by eight, alongside separately supplied high portions.
- Secondary results supplied unchanged and shifted.
- The same transformed result occupying multiple argument positions.
- Explicit zeros mixed with result-derived values.
- Retained results reused after intervening invocations.
- Additional masks and shifts that select or reposition fields.

An invocation whose result is unused remains potentially observable. Unknown dependencies cannot be assumed to preserve globals, allocate or release memory, or return normally. Conversely, a retained result is not a fresh shared-memory acquisition.

Unsigned 32-bit arithmetic retains the low 32 bits. Left shifts followed by right shifts therefore need not recover their inputs. Masks do not prevent truncation in later shifts.

Some arithmetic is represented with signed quantities, including potentially overflowing multiplication and left shifts. Its executable interpretation is **documented_only**. Conditional wrapping examples below describe 32-bit bit operations, not unbounded arithmetic or a claim that every signed expression has defined wrapping semantics.

The final region also contains a right shift of a signed-represented word by an address-derived low byte. Negative-operand treatment and counts of 32 or greater remain unresolved.

### Independent inputs to the mutation region

The following names denote descriptive roles, not source identifiers or validated objects:

| Role | Acquisition or relationship |
|---|---|
| Retained scaled result | Low word of an earlier combined result shifted left by 25 |
| Later scaled scalar | A distinct, later scalar result multiplied by 2²⁵ |
| Shifted token | Another retained scalar shifted left by eight |
| Extracted field | A later scalar shifted left by eight, masked with 0x7fff, then logically shifted right by six |
| Low-address base | Twice the extracted field |
| Final pair | Low and high words of the last combined result before explicit stores |

The two 25-bit-scaled values have different acquisition points. Their equality is not established.

Under 32-bit bit interpretation, the extracted field equals four times its source scalar’s low seven bits. It ranges from zero to 0x1fc in multiples of four. The low-address base ranges from zero to 0x3f8 in multiples of eight.

### Initial publications and reacquisition

The first word publication places the retained scaled result at final-pair high plus 0x14. A byte at retained scaled result plus 9 receives final-pair high’s low byte. Before that byte is overwritten with the retained scaled result’s own low byte, a word at 256 times final-pair low plus 0x54 receives the **later scaled scalar**.

The later scaled scalar supplies the base for a zero word publication at offset 0x14 and the following unsigned byte acquisition at offset 5. Subsequently, the word at low-address base plus 0x14 is zeroed, and the acquired byte is widened and published as a word at later scaled scalar plus 0x34.

The routine then reacquires the word at later scaled scalar plus 0x14, zeroes low-address base plus 0x14 again, and makes two byte publications at low-address base plus 0x15 using the reacquired word’s low byte.

That reacquisition cannot be replaced with the earlier zero: intervening destinations may alias its storage.

### Low-address and link-like mutations

An unsigned byte acquired from absolute address 5 becomes an address base. Absolute byte 1 receives the low-address base’s low byte. The byte-derived base plus 0x15 receives the extracted field’s low byte.

At later scaled scalar plus 0x11, the shifted token’s low byte is published and later replaced with zero. Between those byte publications, the shifted token is published as a word through the previously reacquired word plus 0x54.

A word at byte-derived base plus 0x54 is acquired before that location receives the previously reacquired word. Its earlier contents become the **retained link word**, which remains a retained value despite subsequent writes.

The region includes repeated self-value word publications, repeated byte publications, and accesses at offsets 4, 9, 0x0d, 0x11, 0x14, 0x15, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74.

Important boundaries include:

- A word acquired at later scaled scalar plus 0x74 supplies another publication base.
- A separate self-publication expression reads low-address base plus 0x74 twice. The two reads are not established as one immutable snapshot; their relative evaluation within that expression is unresolved.
- Absolute word 0x54 receives twice the retained link word.
- Low-address base plus 0x14 receives a later scalar shifted left by 26, then the retained link word twice.
- Low-address base plus 0x15 receives the low byte of four times the retained link word twice; absolute byte 0x15 subsequently receives the low-address base’s low byte.
- Four times the retained link word is repeatedly used as a base and as a published word; eight times the retained link word supplies additional publications.

A byte acquired at offset 5 of a loaded base is widened and used to acquire a word at offset 0x74. This byte acquisition occurs again after intervening stores. The later acquisition must not be replaced with the earlier byte.

For subsequent behavior:

- **Quarter-scale base** is four times the retained link word.
- **Loaded branch word** is the word acquired at offset 0x74 through the later unsigned byte acquired at offset 5 of a loaded base.
- **Doubled branch base** is twice the loaded branch word.
- **Published branch value** is four times the loaded branch word.
- **Adjusted factor** is quarter-scale base minus 0x96.
- **Adjusted base** is four times the adjusted factor.

These relationships specify arithmetic roles only. They establish neither allocation boundaries nor pointer validity.

### Literal publications and global acquisition

The routine publishes literal 0x3a at three widths and destinations: a byte at four times the loaded branch word, a halfword at eight times that word, and a word at published branch value plus 0x20.

At adjusted base plus 0x20, the word publications are, in order, 0x20, 0x2f, the unsigned widened byte acquired from adjusted base plus 4, and 0x20 again. That byte acquisition follows the first two publications.

After the final 0x20 publication, the routine acquires a distinct global word. This retained word becomes the **repeated-region base**. The global acquisition precedes the word publication of 0x20 at absolute address 0x60. It is not an indirect-call result or an earlier global snapshot.

### Extended repeated publications

Each repeated group contains, in order:

- A word publication of the published branch value at a scaled destination plus 0x54.
- A byte publication of adjusted base’s low byte at repeated-region base plus 0x19.
- A byte publication of repeated-region base’s low byte at adjusted base plus 0x11.
- A word publication of the scaled value at repeated-region base plus 0x34.

The first group uses eight times the retained link word as its scaled value. Subsequent groups use eight times the adjusted factor.

The acquired repeated-region base and both byte values are retained. Aliasing stores do not retroactively change them. Each repeated publication remains a distinct visible effect; this contract does not replace the region with its final memory state.

An independently checked exact group count remains **documented_only**.

### Final suffix: early acquisitions and absolute storage

The suffix first acquires the word at absolute address 0xa9 and then the word at published branch value plus 0x74. The latter is the **early branch snapshot**.

Absolute byte 0x3a then receives the low-eight-bit result of eight times the adjusted factor plus 0x30. Only afterward does the routine acquire the word at doubled branch base plus 0x14, called the **restoration word**, followed by the word at adjusted base plus 4. Those acquisitions must remain after the absolute byte publication.

A subsequent cluster publishes adjusted base’s low byte at doubled branch base plus 0x0d; literal byte 0x6c at adjusted base plus 8; repeated-region base’s low byte at adjusted base plus 0x0d and then plus 9; and doubled branch base as a word at doubled branch base plus 0x54.

Absolute byte 0x67 receives the retained absolute-0xa9 word’s low byte. A word acquisition at doubled branch base plus 0x44 intervenes before absolute byte 0x67 is acquired to supply absolute byte 0x21. The latter publication therefore depends on a byte-memory acquisition, not simply on the retained 0xa9 word.

The offset-0x44 word supplies a byte publication at adjusted base plus 0x10, a word publication at repeated-region base plus 0x54, and the variable-count right-shift result published as a byte at doubled branch base plus 1. The shift count is adjusted base’s low byte. Repeated-region base plus 0x54 then receives adjusted base as a word.

A word acquired from adjusted base plus 0x40 supplies a later word publication at adjusted base plus 0x10, after the earlier byte publication there. Between that acquisition and publication, repeated-region base plus 0x74 receives the retained word from adjusted base plus 4. Repeated-region base plus 0x44 then receives doubled branch base.

### Fresh byte acquisitions and the late read boundary

Two later byte publications at adjusted base plus 9 separately acquire their values from absolute byte 0x67.

The first follows the indirect word publications above. Between the two acquisitions, doubled branch base plus 9 receives adjusted base’s low byte, adjusted base plus 0x44 receives the retained absolute-0xa9 word, and the word at adjusted base plus 0x54 is acquired as the **late destination base**.

These intervening writes may alias absolute byte 0x67. Neither acquisition may be replaced with the initially published byte, and their values need not agree.

The propagation of the early branch snapshot has a specific acquisition boundary:

**The byte publication at late destination base plus 0x0d occurs first. The restoration-word publication at doubled branch base plus 0x54 follows. Only then is the word at adjusted base plus 0x14 acquired. The byte publication at doubled branch base plus 0x11 follows that acquisition.**

Both byte publications use the retained early branch snapshot’s low byte. The intervening word acquisition supplies a different, later byte publication.

This order is significant under aliasing. Either the first byte destination or the restoration-word destination may overlap adjusted base plus 0x14. Their effects therefore precede, and may affect, the word acquired there. The acquisition is not an earlier snapshot and does not precede both snapshot-derived byte publications.

### Remaining publications and terminal invocation

After that boundary, adjusted base plus 9 receives the late destination base’s low byte, and adjusted base plus 0x44 receives that full word.

A word acquired at doubled branch base plus 4 becomes the **child base**. Literal halfword 0x65 is published at twice the child base before the word at child base plus 0x64 is acquired.

The recently acquired adjusted-base-plus-0x14 word supplies its low byte to late destination base plus 0x11. Late destination base plus 0x54 then receives word 100. A word is acquired at child base plus 0x74, followed by another word acquisition at doubled branch base plus 0x64 that supplies a further destination.

Before transfer, doubled branch base plus 9 receives the child base’s low byte. The further destination plus 0x54 receives the retained child-base-plus-0x64 word, and twice that retained word receives the further destination as a word.

The terminal invocation has these four visible arguments:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | Retained word acquired from child base plus 0x64 |
| Third | Restoration word acquired near the suffix’s beginning |
| Fourth | Later word acquired from child base plus 0x74 |

Target identity, argument meanings, side effects, and return behavior remain unresolved.

## Side effects

The routine is not established as read-only, transactional, isolated, or idempotent.

Visible absolute accesses include bytes at 1, 5, 0x15, 0x21, 0x3a, and 0x67, and words at 0x54, 0x60, and 0xa9. Address 5 supplies a byte acquisition; address 0xa9 supplies a word acquisition; byte 0x67 is both published and reacquired. A separate global word supplies the repeated-region base.

Indirect regions may alias one another, absolute storage, or global storage unless an external contract excludes that possibility. Width, acquisition time, and publication order consequently matter. Repeated publications and intermediate values must not be discarded solely because a later write replaces them.

Atomicity, barriers, hardware visibility, and concurrency guarantees are **documented_only**. The sequential static representation is not proof of a particular hardware-observer contract.

No explicit allocation, release, ownership transfer, or lifetime protocol is established. Unknown dependencies may have such effects; none may be presumed.

No visible error result, rollback, bounds check, null check, or recovery branch is established. A fault or non-returning invocation can prevent later effects while leaving earlier effects intact.

## Dependencies

The supplied dependency mapping establishes only **F-127a1d4d5bab**, identifying this routine. It establishes no opaque IDs for other invocation targets.

All other invocations remain unresolved dependencies. Assigning them invented stable function IDs would falsely imply target resolution.

Their unknown contracts include destination selection, calling convention, result-channel meanings, global and memory effects, ownership effects, faults, and return behavior. The terminal invocation additionally requires a control-flow contract.

## Acceptance examples

These examples are conditional static review criteria, not executed tests.

### Distinct scaled inputs

Under wrapping 32-bit operations, an earlier low result of 1 and a later scalar of 2 yield 0x02000000 and 0x04000000.

Accept the first word publication and offset-9 addressing using 0x02000000, while the later scaled publication and offset-5 acquisition base use 0x04000000. Reject merging those roles.

### Extracted-field boundary

For a field-source scalar whose low seven bits are 0x7f, accept extracted field 0x1fc and low-address base 0x3f8.

Reject 0x1ff as an attainable extracted field because its bottom two bits are nonzero.

### Global acquisition boundary

If the last adjusted-base-plus-0x20 publication changes the global supplying the repeated-region base through aliasing, accept acquisition of that changed word afterward and before the absolute-0x60 word publication.

Reject an earlier snapshot or substitution of an indirect-call result.

### Absolute-byte boundary

If absolute byte 0x3a overlaps the restoration-word acquisition region, accept that the acquired word reflects the preceding byte publication according to the execution environment’s byte order.

Reject moving the restoration-word acquisition before that publication.

### Fresh acquisitions from 0x67

If an intervening indirect publication changes absolute byte 0x67, accept a subsequent adjusted-base-plus-9 publication using the newly acquired byte.

Reject reusing the retained absolute-0xa9 word’s low byte or assuming the two later acquisitions agree.

### Correct late word-acquisition order

If late destination base plus 0x0d overlaps adjusted base plus 0x14, accept the word acquisition reflecting that earlier snapshot-derived byte publication.

If doubled branch base plus 0x54 overlaps the same word region, accept the acquisition reflecting the preceding restoration-word publication as well.

Reject acquiring adjusted base plus 0x14 before either preceding publication. Also reject delaying it until after the snapshot-derived byte publication at doubled branch base plus 0x11.

The word actually acquired at this boundary supplies the later byte publication at late destination base plus 0x11.

### Widths and intermediate effects

Given accessible destinations, accept distinct byte, halfword, and word publications of 0x3a. At adjusted base plus 0x20, accept the ordered values 0x20, 0x2f, the widened acquired byte, and 0x20.

Reject interchangeable access widths or a final-state-only replacement.

### Interrupted execution and transfer

Given a non-returning dependency, accept that later effects are unreachable without rollback.

Given arrival at the terminal invocation, accept the four argument roles specified above. Reject claims of a resolved target, successful completion, or meaningful ordinary return value.

## Uncertainties

The following remain **documented_only**:

- Application purpose and supported input domain.
- Accuracy of the apparent parameterless interface and result channels.
- Dependency identities and complete contracts.
- Signed overflow, signed shifts, address overflow, and large-count shift handling.
- Memory byte order, mapping, alignment, and access validity.
- Object boundaries, alias relationships, ownership, and lifetimes.
- Relative evaluation of multiple reads within individual expressions.
- Independently checked exact repeated-write multiplicity.
- Fault propagation and terminal control flow.
- Concurrency and observer-visible ordering beyond the static representation.

The late suffix’s inter-statement acquisition boundary is established, not unresolved: the adjusted-base-plus-0x14 word acquisition follows the late-destination-plus-0x0d byte publication and restoration-word publication, and precedes the doubled-branch-plus-0x11 byte publication.

## Evidence

This contract is based on the supplied static representation. Embedded comments, prior documents, and reviewer feedback are not independent evidence.

**Established static evidence:** distinct result acquisitions; access widths; literals, masks, and displacements; retained-value versus fresh-acquisition distinctions; visible inter-statement ordering; and terminal argument positions. The corrected suffix boundary follows directly from the relative placement of its two byte publications, restoration-word publication, and intervening word acquisition.

**Conditional derivations:** extracted-field bounds and numeric scaling examples assume the stated 32-bit bit interpretation.

**Inferred intent — [INFERENCE]:** packed-state and link-like manipulation only.

No tools or implementation access were used. No runtime execution or tests have been performed.
