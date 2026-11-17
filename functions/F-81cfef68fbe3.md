<!-- Trusted provenance: F-81cfef68fbe3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-81cfef68fbe3

## Purpose

F-81cfef68fbe3 performs an ordered sequence of indirect dependency invocations and integer transformations, then mutates memory through fixed, computed, and loaded addresses before making an unresolved terminal invocation.

No application-level purpose is established. Printable constants do not establish text output. Address-like values and linked accesses do not establish allocation, object construction, serialization, or ownership transfer.

This contract separates:

- **Established static evidence:** operations, widths, data dependencies, and ordering visible in the supplied representation.
- **[INFERENCE]:** consequences requiring an explicitly stated arithmetic or memory model.
- **documented_only:** unresolved behavior that cannot support an unconditional executable requirement.

All later behavior is conditional on preceding invocations returning and preceding accesses succeeding. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface accepts four positional inputs and declares no result.

| Position | Represented width | Use |
|---|---:|---|
| First | 32 bits | Passed unchanged to F-DEP-01. |
| Second | 32 bits | Passed unchanged to F-DEP-01. |
| Third | 32-bit | Passed unchanged to F-DEP-01. |
| Fourth | Signed 32 bits | Shifted left by 25 positions before being passed to F-DEP-01. |

The first three inputs have no established pointer, handle, encoding, or numeric interpretation. The visible body does not directly dereference them.

### Implicit inputs and outputs

Additional inputs include incoming control-link state, an incoming working-register value, dependency results, secondary result words, fixed-location state, and memory subsequently used for address selection.

The entry target is derived by adding `0xffe` to incoming control-link state. Equal explicit arguments therefore do not establish equal behavior.

There is no represented return value, status code, or structured error. Observable outputs are invocations, memory reads, memory writes, and effects performed by dependencies.

Execution requires accessible targets, compatible calling conventions, usable result values, and valid memory for each represented access width. Alignment support, low-address accessibility, and memory attributes are environmental preconditions; this function does not validate them.

A return follows the terminal invocation in the representation. Whether the underlying terminal transfer returns is **documented_only**.

## Behavior

### Dependency phase

A long straight-line sequence of indirect invocations precedes the first explicit data-memory store. No visible conditional branch or iteration selects among alternative paths.

F-DEP-01 receives the four entry operands described above. Intermediate invocation sites, collectively designated F-DEP-02, consume primary results, secondary results, halves of represented 64-bit results, retained earlier values, zeros, and transformed bit fields.

F-DEP-02 denotes an ordered collection of sites, not one established shared implementation. Calls with unused visible results remain observable. Calls with no represented operands are not proven zero-input interfaces: implicit machine-state dependencies remain unresolved.

The sequence must not be replaced with a reordered or final-result-only computation. No dependency is established as pure, globally preserving, memory preserving, or ownership preserving.

### Arithmetic and storage widths

The principal scalar width is 32 bits. Some dependency results are represented as 64-bit quantities with separately consumed lower and upper 32-bit words.

Transformations include low-field masks of widths 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11, as well as individual-bit selection. Other operations select bits 10–19 of a word or bits 11–17 of an upper result word.

Unsigned 64-bit right shifts by 34, 36, and 44 positions occur before narrowing. A full unsigned 64-bit shift by 44 yields the upper word shifted logically right by twelve; it is not a 32-bit shift by 44.

Unsigned right shifts are logical. Signed left shifts, signed scaling, overflow, address wrapping, and the terminal signed right shift require execution semantics not fully established here.

**[INFERENCE]:** Under a 32-bit machine-word model retaining the low 32 bits:

- A left shift by 25 retains contributions only from the original low seven bits.
- Successive left shifts by 25 and seven yield zero.
- Successive left shifts by 28 and 28 yield zero.
- A word shifted left by eight and masked with `0x7fff` ranges from zero through `0x7f00` in multiples of `0x100`.
- Shifting that masked value logically right by six yields zero through 508 in multiples of four. Doubling yields zero through **1016 inclusive**, in multiples of eight.

These consequences do not establish language-level semantics for every signed expression.

Byte stores retain eight bits; halfword stores retain sixteen. Explicit unsigned byte reads are zero-extended when published as words. This does not determine the storage width or conversion behavior of the separate fixed-location read associated with `0x05`.

### Initial memory publication

The following descriptive names identify captured values, not objects or private source identifiers:

| Name | Meaning |
|---|---|
| Packed value | An earlier primary result shifted left by 25. |
| Scaled base | Another primary result multiplied by \(2^{25}\). |
| Half-small value | The masked-and-shifted value conditionally ranging through 508. |
| Small base | Twice the half-small value, conditionally ranging through 1016. |
| Final pair | The lower and upper words of the last dependency result preceding direct stores. |
| Retained word | Another earlier primary result shifted left by eight. |

Initial access ordering is:

| Interval | Required relative ordering |
|---|---|
| Cross-address publication | Word publication of the packed value at final-upper-word plus `0x14`; byte publication of the upper word at packed-value plus nine; word publication of the scaled base at 256 times final-lower-word plus `0x54`. |
| Overwrite and reset | The packed-value-plus-nine byte is overwritten with the packed value’s low byte; scaled-base plus `0x14` is cleared as a word; an unsigned byte is acquired from scaled-base plus five. |
| Byte-to-word publication | Small-base plus `0x14` is cleared; the acquired byte is stored as a zero-extended word at scaled-base plus `0x34`; scaled-base plus `0x14` is read as a word. |
| Repeated narrowing | Small-base plus `0x14` is cleared again; the low byte of the preceding word read is written to small-base plus `0x15` twice consecutively. |

The word read after clearing remains an access. Its result is zero only under suitable ordinary-memory assumptions, including absence of intervening overlapping writes, external mutation, and read-side effects.

### Fixed-location conversion and linked updates

After the consecutive byte stores, state associated with fixed address `0x05` is read and converted to an unsigned 32-bit word. The converted value supplies an address base.

Its storage declaration is absent. The read width, original signedness, and extension behavior are **documented_only**. Neither a byte access nor an address base below 256 is established.

That acquisition precedes the byte write at fixed address `0x01`, which receives the small base’s low byte.

Subsequent accesses publish the half-small value’s low byte at converted-base plus `0x15`, publish the retained word’s low byte at scaled-base plus `0x11`, and publish the retained word as a word through the earlier reset-read value at offset `0x54`. The scaled-base-plus-`0x11` byte is then cleared.

A word is captured from converted-base plus `0x54` before that location is replaced with the earlier reset-read value. This captured word supplies later scaling and addressing.

Further linked updates include:

- Acquisition through scaled-base plus `0x74`, followed by word publications through that loaded address.
- Two represented reads through small-base plus `0x74` for a self-valued word publication at offset `0x54`; identical address expressions do not establish one cached read.
- Repeated self-valued word stores at captured-word plus `0x24`, interleaved with byte stores at another loaded base plus nine.
- Word publications at small-base plus `0x14`, including two publications of the captured word.
- A fixed word store at `0x54` containing twice that captured word.
- Repeated byte stores at small-base plus `0x15`, and a fixed byte store at `0x15`.

Later operations scale retained loaded values by two, four, and eight; follow byte-selected addresses at offset `0x74`; and perform word acquisitions at offsets four, `0x64`, and `0x74`. Explicit unsigned byte reads at offset five participate in subsequent address selection and are not interchangeable with the unresolved fixed-location conversion.

These accesses establish no independent-field record layout. Bases may alias, and byte writes may overlap word accesses.

### Constant publication and persistent-base capture

A later loaded word determines three destinations receiving `0x3a`: a byte at four times that word, a halfword at eight times it, and a word at four times it plus `0x20`.

For the remainder:

- **Linked base** means twice that loaded word.
- **Published word** means four times that loaded word.
- **Adjusted value** means an earlier four-times-scaled value reduced by `0x96`.
- **Computed base** means four times the adjusted value.

Computed-base plus `0x20` receives four word stores, in order: `0x20`, `0x2f`, a zero-extended byte freshly acquired from computed-base plus four, and `0x20` again. The byte acquisition occurs between the second and third stores.

Only after these stores does the function acquire fixed-location state supplying the **persistent base**. Only after that acquisition does it store word `0x20` at fixed address `0x60`. The persistent base is retained rather than represented as freshly acquired during each subsequent publication.

The repeated publication phase orders each group as follows:

- A word publication of the published word at an eight-times-scaled destination plus `0x54`.
- A byte publication of the computed base at persistent-base plus `0x19`.
- A byte publication of the persistent base at computed-base plus `0x11`.
- A word publication of the eight-times-scaled value at persistent-base plus `0x34`.

The initial group uses the earlier unadjusted loaded value for eight-times scaling. Subsequent groups use the adjusted value.

Repeated writes remain distinct represented effects. Their apparent redundancy in ordinary memory does not establish permission to collapse them in a trace-sensitive environment.

### Terminal captures and fresh fixed-location reads

The terminal phase first captures the word at fixed address `0xa9`, then captures the word at published-word plus `0x74`. Both captures precede the byte write at fixed address `0x3a`.

That byte receives the narrowed result of eight times the adjusted value plus `0x30`. The function then captures words at linked-base plus `0x14` and computed-base plus four.

Before publishing the low byte of the earlier `0xa9` capture at `0x67`, it writes:

- The computed base’s low byte at linked-base plus `0x0d`.
- Byte `0x6c` at computed-base plus eight.
- The persistent base’s low byte at computed-base plus `0x0d` and then plus nine.
- The linked base as a word at linked-base plus `0x54`.

The following intervals preserve the significant capture and aliasing boundaries:

| Interval | Observable ordering |
|---|---|
| Initial fixed-byte publication | The low byte of the `0xa9` capture is written to `0x67`; linked-base plus `0x44` is read as a signed word; a fresh byte read of `0x67` supplies the write at `0x21`. |
| Linked-word publication | The loaded signed word’s low byte is written at computed-base plus `0x10`; the word is published at persistent-base plus `0x54`; a shifted byte is written at linked-base plus one; persistent-base plus `0x54` is overwritten with the computed base. |
| Captured-word restoration | Computed-base plus `0x40` is read; the earlier computed-base-plus-four capture is published at persistent-base plus `0x74`; the new capture is stored as a word at computed-base plus `0x10`; the linked base is published at persistent-base plus `0x44`. |
| First computed-address copy | A fresh byte read of `0x67` supplies computed-base plus nine; the computed base’s low byte is written at linked-base plus nine; the original `0xa9` capture is stored as a word at computed-base plus `0x44`. |
| Second computed-address copy | A word is captured from computed-base plus `0x54`; another fresh byte read of `0x67` supplies another write at computed-base plus nine. |

There are three distinct terminal reads of `0x67`. None is specified as reuse of its initially published value.

The signed right shift uses the word acquired from linked-base plus `0x44`. Its count is the computed base’s low eight bits. Negative-operand behavior and counts of 32 or greater remain **documented_only**.

### Terminal tail: retained bases and ordering boundaries

Two loaded address values must remain distinct:

| Descriptive name | Acquisition | Subsequent uses |
|---|---|---|
| Retained tail base | The word captured from computed-base plus `0x54`, before the second computed-address copy from `0x67`. | Byte destinations at offsets `0x0d` and `0x11`; word destination at `0x54`; publications into the computed-base region. |
| Later tail base | A later word read from linked-base plus four. | Halfword destination at twice this base; word reads at this base plus `0x64` and plus `0x74`; a later narrowed publication at linked-base plus nine. |

After the second computed-address copy from `0x67`, the low byte of the earlier published-word-plus-`0x74` capture is written at **retained-tail-base plus `0x0d`**. Next, linked-base plus `0x54` is restored from the earlier linked-base-plus-`0x14` capture. Next, computed-base plus `0x14` is captured as a word. **Only after that read** is the same captured byte written at **linked-base plus `0x11`**.

Those two byte stores are not consecutive. The restoration and intervening word capture separate them. This ordering remains significant when the first byte destination overlaps the restored word or either byte destination overlaps the computed-base-plus-`0x14` capture.

The computed-base region then receives the retained tail base’s low byte at offset nine and the retained tail base as a word at offset `0x44`, before the later tail base is acquired from linked-base plus four.

The remaining access boundaries are:

- Halfword `0x65` is stored at twice the **later tail base**, before the word read at later-tail-base plus `0x64`.
- After that read, the low byte of the captured computed-base-plus-`0x14` word is stored at **retained-tail-base plus `0x11`**, followed by word 100 at **retained-tail-base plus `0x54`**.
- The terminal offset-`0x74` word is then captured from the **later tail base**, not the retained tail base.
- Linked-base plus `0x64` is read before the later tail base’s low byte is written at linked-base plus nine.
- Through the newly loaded linked-base-plus-`0x64` value, offset `0x54` receives the earlier later-tail-base-plus-`0x64` result. A word at twice that earlier result then receives the newly loaded value.

F-DEP-03 receives four represented operands, in order: `0x72`, the later-tail-base-plus-`0x64` capture, the earlier linked-base-plus-`0x14` capture, and the later-tail-base-plus-`0x74` capture. These are retained values, not guaranteed final contents of their acquisition locations.

## Side effects

The direct footprint includes 8-bit, 16-bit, and 32-bit accesses through computed and loaded addresses. Byte-relative offsets include one, five, eight, nine, `0x0d`, `0x10`, `0x11`, `0x15`, and `0x19`. Word-relative offsets include four, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, and `0x74`. Halfword destinations include scaled addresses without a field offset.

Established fixed-address effects include byte writes at `0x01`, `0x15`, `0x21`, `0x3a`, and `0x67`; word writes at `0x54` and `0x60`; a word read at `0xa9`; and fresh terminal byte reads at `0x67`.

No atomicity, transaction, synchronization, rollback, independent-object layout, or interruption safety is established. Overlapping accesses must be interpreted using the applicable byte order and memory model.

No allocation, release, reference-counting, or ownership-transfer contract is established. Dependencies may affect storage lifetime; preservation cannot be assumed.

There is no visible failure branch for invalid targets, inaccessible memory, overflow, or malformed state. Failure may leave earlier effects in place. Concrete fault behavior is **documented_only**.

## Dependencies

| Opaque ID | Established boundary |
|---|---|
| F-DEP-01 | Entry invocation derived from incoming control-link state; receives the first three inputs unchanged and the transformed fourth input. |
| F-DEP-02 | Ordered intermediate invocation sites supplying primary, secondary, and paired results; no shared identity is established. |
| F-DEP-03 | Unresolved terminal invocation receiving the four retained operands specified above. |

Actual identities, complete calling conventions, global preservation, memory lifetime effects, exceptions, blocking, reentrancy, and return behavior remain **documented_only**.

## Acceptance examples

These are conditional requirements, not executed tests.

1. **Entry transformation.** Under low-32-bit shift semantics, fourth-input values one and 129 both supply `0x02000000` to F-DEP-01. This does not require matching complete executions.

2. **Small-base boundary.** A contributing result with low seven bits `0x7f` yields masked value `0x7f00`, half-small value 508, and small base 1016. Excluding 1016 is incorrect.

3. **Unsigned acquisition.** An explicit offset-five byte read yielding `0xe1` supplies word value `0x000000e1` to its corresponding word publication. This does not determine the fixed `0x05` read width.

4. **Persistent capture.** Its acquisition follows all four computed-base-plus-`0x20` stores and precedes the fixed word store at `0x60`.

5. **Fresh fixed-byte reads.** If an intervening overlapping write changes `0x67`, each subsequent terminal copy uses its own newly read byte, not the initially published byte.

6. **Separated byte stores.** If retained-tail-base plus `0x0d` overlaps linked-base plus `0x54`, the captured-byte write precedes restoration of that word. The computed-base-plus-`0x14` capture follows restoration and precedes the byte write at linked-base plus `0x11`.

7. **Read-before-overwrite boundary.** If linked-base plus `0x11` overlaps the word at computed-base plus `0x14`, the captured word reflects memory before that byte write. Moving both captured-byte stores ahead of the read is unacceptable.

8. **Distinct tail bases.** If the retained tail base is `0x1000` and the later tail base is `0x2000`, with valid accessible destinations, the captured byte goes to `0x1011` and word 100 to `0x1054`. Halfword `0x65` goes to `0x4000`; the terminal word acquisitions use `0x2064` and `0x2074`.

9. **64-bit extraction.** An upper result word of `0x12345678` yields `0x00012345` after the full unsigned pair is shifted right by 44 and narrowed.

10. **Terminal operands.** Later overlapping writes do not replace the retained captures passed to F-DEP-03. Its first represented operand is `0x72`; subsequent return remains unresolved.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended data model.
- Complete intermediate invocation contracts and implicit inputs.
- Signed arithmetic, overflow, address wrapping, and exceptional shift behavior.
- Storage width and pre-conversion interpretation of state associated with `0x05`.
- Byte order, alignment, low-address validity, memory attributes, concurrency, and alias relationships.
- Ownership, lifetime, dependency effects, and terminal return behavior.
- Fidelity of unusual represented operations to executable behavior.

These gaps do not authorize added validation, cached replacements for fresh reads, collapsed repeated effects, or reassignment of accesses between retained address values.

## Evidence

The behavioral claims above derive solely from static inspection of the supplied representation. Prior prose, comments, and reviewer statements are not independent behavioral authority.

The terminal ordering is established by the intervening restoration and word acquisition between the two captured-byte publications. The tail-base distinction is established by the separate acquisitions and their different destination and read relationships.

The arithmetic bounds follow from the represented masks and widths. Missing storage information prevents resolving the fixed `0x05` conversion.

No implementation was executed, no runtime trace was collected, and no acceptance example was tested. No snapshot hash is asserted.
