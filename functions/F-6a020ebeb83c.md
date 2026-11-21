<!-- Trusted provenance: F-6a020ebeb83c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6a020ebeb83c

## Purpose

F-6a020ebeb83c invokes an ordered series of unresolved dependencies, transforms selected results, performs shared-memory reads and writes, and reaches a terminal indirect invocation.

No application-level purpose is established. Repeated offsets, address-like values, and character-like constants do not establish objects, collections, text, or an initialization protocol.

This contract distinguishes:

- **Established static evidence:** represented operations, value dependencies, explicit access widths, and sequencing.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** behavior not resolved by the supplied representation.

All behavior is conditional on the representation accurately describing execution and on execution reaching the relevant operation.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters or return value. No status result or caller-supplied output buffer is established.

Implicit inputs include incoming control-link state, dependency results, additional outgoing register values, shared globals, and calculated-address memory.

The terminal invocation has four represented argument words, beginning with `0x72`. A return is represented afterward; whether the indirect destination returns is unresolved.

### Widths and arithmetic

An explicitly typed byte, halfword, or word dereference establishes an 8-, 16-, or 32-bit access respectively. Selected dependency results are represented as 64-bit values whose low and high 32-bit words are consumed separately. Explicit unsigned-byte conversions zero-extend; explicit byte and halfword narrowing retain the low eight and sixteen bits.

**These rules do not establish the access widths of declaration-less globals.** Assignment of a narrowed value to a global establishes the supplied value’s width, not the destination’s storage width. Reading a global into a 32-bit saved value or through an 8-bit destination likewise does not establish the global load width.

Arithmetic examples use conditional 32-bit bit-vector semantics: addition, subtraction, multiplication, and left shifts retain the low 32 bits; unsigned right shifts insert zeros. This model does not resolve signed overflow or certify executable behavior for signed expressions.

A late signed right shift uses a count taken from a calculated word’s low eight bits. Its behavior for counts of 32 or greater, including any count masking, is **documented_only**.

Endianness, alignment, atomicity, and memory-mapped access semantics are unresolved.

### Preconditions and ownership

Normal traversal requires callable indirect destinations and permitted access to every reached memory location. No visible validation establishes bounds, alignment, non-null addresses, or writable storage.

No allocation, release, retention, reference counting, or ownership transfer is established. Address-like arguments do not establish lifetime. Unknown dependencies must not be assumed to preserve globals or own or release memory.

## Behavior

### Dependency-driven preparation

A long straight-line series of indirect invocations precedes the direct mutation region. The first target depends on incoming control-link state plus `0xFFE`; subsequent targets use fixed location-derived calculations. Concrete dependency identities remain unresolved.

Invocation order matters even where no arguments are displayed or the primary result is unused. Calls can affect shared memory or additional register values subsequently consumed.

Represented arguments include shifted results; one, two, or three zero words; independently transformed halves of 64-bit results; extracted fields; saved additional-register outputs; and equal transformed values in multiple argument positions. Matching argument forms do not establish matching callees.

Important transformations include:

| Transformation | Retained information under the arithmetic model |
|---|---|
| Mask `0xFFFFFF`, then unsigned right shift 8 | Original bits 8–23 |
| Mask `0x3FFFF`, then unsigned right shift 11 | Original bits 11–17 |
| Mask `0x1FFFF`, then unsigned right shift 4 | Original bits 4–16 |
| Left shift 8, mask `0x7FFF`, then unsigned right shift 6 | Four times the original low seven bits |
| Mask `0xFFFFFF`, then unsigned right shift 12 | Original bits 12–23 |

Shifts apply to already transformed words. Two successive left shifts by 14 leave only the original low four bits at positions 28–31. Left shifts by 14 and then 23 yield zero under the stated model, not a shift by five. Independently shifting result halves is not a combined 64-bit shift.

Several saved values are explicitly zeroed before later transformations; their derived arguments remain zero under this model.

### Opening direct effects

The following names describe provenance, not object types:

- **Packed base:** a selected dependency-result low word shifted left by 25.
- **Scaled base:** a later returned word multiplied by `0x02000000`.
- **Small index:** four times another result’s low seven bits.
- **Final pair:** the last represented 64-bit result before direct stores.
- **Shifted payload:** a saved returned word shifted left by eight.

Packed base and scaled base each retain only seven input bits under 32-bit truncation.

Opening effects publish packed base as a word at final-pair high word plus `0x14`, then write that high word’s low byte at packed base plus `9`. Scaled base is published at final-pair low word multiplied by `0x100`, plus `0x54`. Packed base plus `9` is overwritten with packed base’s low byte.

The routine clears scaled base plus `0x14`, reads a byte at scaled base plus `5`, clears twice small index plus `0x14`, and publishes the zero-extended byte at scaled base plus `0x34`.

A word is read back from scaled base plus `0x14`. The clear at twice small index plus `0x14` is repeated, followed by two byte writes of the readback word’s low byte at twice small index plus `0x15`.

The readback is an actual read. Aliasing and unusual memory semantics prevent unconditionally replacing its result with zero.

### Global-derived addressing and initial rewiring

The global associated with absolute address `0x05` is read and converted to an unsigned 32-bit value, called **converted global base**. Its declaration, access width, and signedness are unresolved. The conversion does not establish a byte load, zero-extension, or a range of `0`–`255`.

Next, a value narrowed to twice small index’s low eight bits is assigned to the global at `0x01`. The actual store width is unresolved. An explicit byte store then places small index’s low byte at converted global base plus `0x15`.

Scaled base plus `0x11` receives shifted payload’s low byte. Shifted payload is published as a word through the earlier readback word at `+0x54`; scaled base plus `0x11` is then cleared by a byte store.

A word saved from converted global base plus `0x54` becomes **anchor**, before that location is overwritten with the earlier readback word. Small index is published at twice small index plus `0x34`.

A word read from scaled base plus `0x74` supplies another destination. Its `+0x54` receives the original scaled-base input shifted left by 26. A nested publication separately reads twice small index plus `0x74` twice: one result supplies the destination component, the other the payload. Their equality and relative evaluation order are not independently established.

Anchor is self-published at anchor plus `0x24`; anchor’s low byte is written at the other destination plus `9`. That destination’s `+0x54` receives shifted payload. The anchor self-publication and destination byte write are repeated.

Twice small index plus `0x14` receives the left-by-26 value. Twice anchor is then assigned to the global at `0x54`, with unresolved store width. Twice small index plus `0x14` subsequently receives anchor twice.

Define **quarter base** as four times anchor and **anchor scale** as eight times anchor. Quarter base’s low byte is written twice at twice small index plus `0x15`. Twice small index, narrowed to eight bits, is assigned to the global at `0x15`; that global’s store width remains unresolved.

### Quarter-base publication and linked reads

Quarter base plus `0x54` first receives shifted payload. Quarter base is published at twice small index plus `0x34`. A fresh word read at twice small index plus `0x74`, multiplied by two, supplies **early partner**. Quarter base plus `0x54` is overwritten with quarter base.

Quarter base plus `0x15` receives anchor scale’s low byte, and early partner is published at anchor plus `0x44`.

An explicit unsigned-byte read at quarter base plus `5`, zero-extended, supplies the base for a word read at `+0x74`. Call that word **first linked destination**, and its saved low byte **linked byte**.

After early partner is published at first linked destination plus `0x44`, the ordered effects are:

- Quarter base’s low byte is written at quarter base plus `0x15`.
- Linked byte is written at quarter base plus `0x0D`.
- Quarter base is self-published at `+0x14`.
- Quarter base is published at `+0x54`.

There is **no intervening `+0x15` write between these last two word stores**.

A word read at quarter base plus `0x64` supplies a destination receiving linked byte at `+0x0D`, then self-publication at `+0x14`. A word read at quarter base plus `4` supplies a **byte-source base**.

Before the first unsigned-byte read at byte-source base plus `5`, early partner plus `9` receives linked byte and early partner plus `0x44` receives anchor scale.

Between the first and second byte reads at that same source location, the routine writes, in order:

- Early partner’s low byte at quarter base plus `9`.
- The first byte result, zero-extended, at first linked destination plus `0x44`.
- Anchor scale at quarter base plus `0x14`.
- Byte-source base’s low byte at early partner plus `0x15`.
- Quarter base at early partner plus `0x44`.

The early-partner `+9` write is before both reads, not between them.

After the second byte read, quarter base plus `0x15` receives anchor scale’s low byte. The second byte result, zero-extended, supplies a fresh word read at `+0x74`. Call that result **linked word**, and twice it **partner base**.

Subsequent writes place byte-source base’s low byte at quarter base plus `0x15`, quarter base at `+0x54`, partner base’s low byte at `+9`, and anchor scale at `+0x14`.

### Literal stores and repeated publication

Literal `0x3A` is stored, in order, as a byte at four times linked word, as a halfword at eight times linked word, and as a word at four times linked word plus `0x20`.

Define **adjusted scale** as quarter base minus `0x96`, and **adjusted base** as four times adjusted scale.

Adjusted base plus `0x20` receives ordered word writes of `0x20`, `0x2F`, a freshly read and zero-extended byte from adjusted base plus `4`, and `0x20`. The byte read occurs after the `0x2F` store and before publication of its result.

Only afterward is a separately stored global read into a 32-bit saved value, **publication base**. Its actual load width and declaration remain unresolved. Then `0x20` is assigned to the global at `0x60`, whose store width is also unresolved. The saved-base read precedes that assignment.

A fixed repeated publication region follows. Each group performs four ordered explicit accesses: a word publication of four times linked word at destination scale plus `0x54`; a byte write of adjusted base at publication base plus `0x19`; a byte write of publication base at adjusted base plus `0x11`; and a word publication of destination scale at publication base plus `0x34`.

The first destination scale is eight times anchor; later groups use eight times adjusted scale. Publication base remains the saved value. No condition-controlled loop is established. The exact repetition count is **documented_only** in this contract; repeated accesses must not be collapsed merely because values match.

### Final snapshots and global accesses

After the repeated region, the global at `0xA9` is read into a 32-bit saved value, **saved absolute value**. This establishes the saved value’s width, **not a 32-bit global load**. A word is then saved from four times linked word plus `0x74`.

Next, a character-derived expression is assigned to the global at `0x3A`. Under the conditional arithmetic model, its low eight bits equal those of eight times adjusted scale plus `0x30`. The global’s actual store width, conversion, and complete stored value remain unresolved.

Only after that assignment are words saved from partner base plus `0x14` and adjusted base plus `4`.

Partner base plus `0x0D` receives adjusted base’s low byte. Adjusted base receives byte `0x6C` at `+8`, then publication base’s low byte at `+0x0D` and `+9`. Partner base is self-published at `+0x54`.

Saved absolute value is narrowed to eight bits and assigned to the global at `0x67`. **This does not establish an 8-bit store.** A word is then read at partner base plus `0x44`. After that read, the global at `0x67` is freshly read and assigned to the global at `0x21`. Neither global access width nor the destination conversion is established; this is not proven to be a byte-to-byte copy.

The loaded partner-base `+0x44` word supplies an explicit byte store at adjusted base plus `0x10`, a word store at publication base plus `0x54`, and a signed-shift-derived byte store at partner base plus `1`. The shift count is adjusted base’s low eight bits. Publication base plus `0x54` is then overwritten with adjusted base.

A word read at adjusted base plus `0x40` precedes publication of the saved adjusted-base `+4` word at publication base plus `0x74`. The newly read word is stored at adjusted base plus `0x10`; partner base is published at publication base plus `0x44`.

### Fresh reads and terminal publication

A second fresh read of the global at `0x67` supplies an explicit byte write at adjusted base plus `9`. Only the consumed byte is established; the global load width is unresolved.

Next, adjusted base’s low byte is written at partner base plus `9`, saved absolute value is published as a word at adjusted base plus `0x44`, and a word is read at adjusted base plus `0x54`, becoming **late destination**.

A third fresh read of the global at `0x67` supplies another explicit byte write at adjusted base plus `9`. These three global reads must remain distinct. Their values need not equal one another or the initially assigned narrowed value.

Late destination plus `0x0D` receives the saved four-times-linked-word `+0x74` word’s low byte. The saved partner-base `+0x14` word is restored at partner base plus `0x54`. A fresh word is read at adjusted base plus `0x14`.

Further writes place the saved `+0x74` word’s low byte at partner base plus `0x11`, late destination’s low byte at adjusted base plus `9`, and late destination as a word at adjusted base plus `0x44`.

A fresh word read at partner base plus `4` supplies **late base**. Halfword `0x65` is written at twice late base before a word is read at late base plus `0x64`.

The earlier adjusted-base `+0x14` result supplies a byte at late destination plus `0x11`; late destination plus `0x54` receives word `100`. Words are then read from late base plus `0x74` and partner base plus `0x64`.

Final stores write late base’s low byte at partner base plus `9`, publish the late-base `+0x64` word through the partner-base `+0x64` destination at `+0x54`, and publish that destination at twice the late-base `+0x64` word.

The terminal invocation receives, in order:

1. `0x72`.
2. The word read at late base plus `0x64`.
3. The saved partner-base `+0x14` word.
4. The word read at late base plus `0x74`.

## Side effects

Observable effects include dependency invocations, distinct reads, intermediate and duplicate stores, mixed-width overwrites, and global assignments—not merely final memory contents.

Names describe provenance, not nonaliasing. Explicit word stores at `+0x14` can overlap bytes at `+0x15`; word stores at `+0x10` can overlap bytes at `+0x11`. Calculated addresses can overlap globals. Missing global declarations prevent determining their full accessed ranges or excluding overlap with neighboring addresses.

No transaction, synchronization, rollback, validation branch, or explicit recovery path is represented. Faults or nonreturning dependencies can leave earlier effects performed. No returned-error guarantee is established.

## Dependencies

| Opaque ID | Scope | Established information |
|---|---|---|
| F-U001 | First indirect invocation | Target depends on incoming control-link state; no displayed arguments |
| F-U002 | Remaining preparation invocations | Ordered occurrence family with varying arguments and selected results |
| F-U003 | Terminal invocation | Four represented argument words beginning with `0x72` |

F-U002 does not imply a shared target. Concrete identities, complete signatures, additional-register conventions, global preservation, ownership, failures, re-entry, and return behavior are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Extraction:** masking `0xAB123456` with `0xFFFFFF`, then shifting unsigned right by eight, yields `0x1234`.

2. **Independent halves:** shifting low word `0x12345678` left by eight yields `0x34567800`; an independently unchanged high word remains unchanged.

3. **Index boundary:** an input with all seven low bits set produces small index `508` and twice small index `1016`.

4. **Global conversion:** if conversion of the `0x05` global yields `0x1234`, its `+0x54` address component is `0x1288`. Truncation to `0x34` is unsupported.

5. **Quarter-base chronology:** following first linked destination `+0x44` publication, the required sequence is quarter-base byte writes at `+0x15` and `+0x0D`, then word publications at `+0x14` and `+0x54`, with no intervening `+0x15` write.

6. **Separated byte reads:** if writes between the two byte-source reads change the source location, distinct results are permitted. Early partner’s `+9` update already occurred before the first read.

7. **Global width:** assigning saved absolute value’s low byte to `0x67` does not justify modeling an 8-bit store or assuming neighboring bytes remain untouched.

8. **Fresh global consumption:** if the second and third `0x67` reads have low bytes `0x34` and `0x56`, the corresponding adjusted-base `+9` byte stores use those values. The first read’s transfer to `0x21` remains subject to unknown global declarations.

9. **Snapshot ordering:** a `0x3A` global assignment that affects the subsequent partner-base `+0x14` read can affect its saved result; that read is not an earlier snapshot.

10. **Failure:** an inaccessible destination has no established rollback or returned-error requirement. Reaching F-U003 establishes its arguments, not successful return.

## Uncertainties

The following remain **documented_only**:

- Representation fidelity, complete calling conventions, and exact preparation-call inventory.
- Exact repeated-publication count.
- Declarations, access widths, signedness, and conversions of declaration-less globals, including `0x01`, `0x05`, `0x15`, `0x21`, `0x3A`, `0x54`, `0x60`, `0x67`, `0xA9`, and the publication-base source.
- Memory mapping, endianness, alignment, concurrency, and access side effects.
- Signed overflow, variable-shift semantics, and unresolved nested-expression evaluation order.
- Address validity, aliasing, ownership, lifetime, and terminal return behavior.

Shared-state publication is an inferred possibility only. No application data structure or lifecycle is established.

## Evidence

The supplied static function representation supports the explicit accesses, saved-value widths, arithmetic dependencies, and represented sequencing. Prior prose, comments, and feedback are not independent behavioral proof.

Explicit dereference types support byte, halfword, and word access claims. Missing global declarations do not support equivalent width claims for named absolute globals, even when surrounding values are narrowed or saved in 32-bit containers.

The represented sequence supports the corrected quarter-base chronology, the early-partner updates before the first byte-source read, the intervening writes between the two reads, and the three distinct later reads of the `0x67` global.

No dependency bodies, executable instruction inspection, global declarations, memory map, or runtime observations were supplied. No runtime tests have been performed.
