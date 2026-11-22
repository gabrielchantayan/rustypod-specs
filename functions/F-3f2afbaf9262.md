<!-- Trusted provenance: F-3f2afbaf9262; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f2afbaf9262

## Purpose

F-3f2afbaf9262 performs an ordered series of indirect invocations, mutates memory through call-derived and memory-derived addresses, and ends with an unresolved four-argument invocation.

Its application-level purpose is not established. Numeric values resembling characters do not establish text processing. Self-valued stores and cross-references do not establish a valid object model, allocation protocol, or ownership transfer.

This contract distinguishes:

- **Established static evidence:** represented access widths, retained-value relationships, and relative ordering.
- **Inferred intent:** possible interpretations that impose no acceptance requirements.
- **documented_only:** unresolved behavior or coverage for which this contract makes no stronger claim.

Continuation requires each reached dependency to return and each reached access to be supported. Descriptions of later effects do not make earlier writes unobservable.

## Interface

### Explicit inputs and output

| Element | Contract |
|---|---|
| First input | Uninterpreted 32-bit word. |
| Second input | Uninterpreted 32-bit word. |
| Declared output | No return value. |
| Explicit failure result | None established. |

The opening invocation receives both explicit inputs unchanged and in their original order. The represented body does not directly dereference them. Their interpretation by dependencies is **documented_only**.

Memory mutations and dependency effects are outputs despite the absence of a return value.

### Additional entry state

Execution also depends on two incoming 32-bit state words, an incoming control-link value, call-associated result state, and memory.

The opening target is constructed by adding 0xffe to the incoming control-link value. Subsequent targets use represented address constructions; their dependency identities are unresolved.

The explicit two-word interface therefore does not describe all execution requirements. Ignoring the additional incoming state is not established to preserve behavior.

### Preconditions

Successful progression requires executable invocation targets and accessible memory at every reached address, with support for the represented access width.

No checks are established for null, low, unaligned, inaccessible, or overlapping addresses; overflow; address wrapping; loaded words subsequently interpreted as addresses; or dependency termination.

No allocation sizes, disjointness guarantees, ownership rights, or preservation obligations are established. These are environmental requirements where needed, not validated restrictions.

## Behavior

### Invocation phase

An extended sequence of indirect invocations precedes the first direct memory write. No explicit conditional branch or loop is represented in this phase. Dependencies may nevertheless mutate memory, fail, or prevent continuation.

The opening invocation has three represented arguments: the two explicit inputs followed by the first additional incoming word logically shifted right by 11.

After two further invocations, a represented 64-bit result supplies the first two arguments of a four-argument invocation:

1. Its low 32-bit word shifted left by eight.
2. Its high 32-bit word unchanged.
3. The second incoming state word logically shifted right by 11.
4. The first incoming state word logically shifted right by 11.

Later invocations consume ordinary word results, numerical halves of 64-bit results, retained earlier results, additional call-associated result words, constants, and masked or shifted derivatives.

An invocation with no represented arguments may have implicit inputs. An unused ordinary result does not justify discarding an invocation: its other effects are unresolved.

Numerical low/high splitting does not establish memory byte order. The complete intermediate signature sequence and implicit calling convention remain **documented_only**.

### Arithmetic and address relationships

The following unsigned extractions have established mathematical bounds:

| Transformation of a 32-bit word | Range |
|---|---|
| Logical right shift by 11 | 0 through 0x1fffff |
| Low 24 bits shifted right by eight | 0 through 0xffff |
| Low 24 bits shifted right by 19 | 0 through 31 |
| Low 18 bits shifted right by 11 | 0 through 127 |
| Low 17 bits shifted right by four | 0 through 8191 |

Under a 32-bit bitvector interpretation, left shifts retain only the low 32 bits. A five-bit field shifted left by 31 therefore retains only its lowest bit; shifting that result left by another 30 produces zero. A word shifted left by 16 has no low 16 bits remaining for subsequent extraction.

Some represented shifts, products, and additions instead involve signed 32-bit quantities. Signed overflow, negative left operands, signed right shifts, and address conversions remain **documented_only**. Bitvector conclusions below are explicitly conditional and do not resolve those semantics.

The following descriptive roles denote retained numerical relationships, not allocated objects:

| Role | Relationship |
|---|---|
| High-shift base | Retained result word shifted left by 25. |
| Independent result | Separate retained signed 32-bit call result. |
| Independent scaled base | Independent result multiplied by 2²⁵. |
| Small base | Result shifted left by eight, restricted to its low 15 bits, then logically shifted right by six. |
| Doubled small base | Twice the small base. |
| Sampled root | Old offset-0x54 word selected through the value read at fixed location 0x05. |
| Fourfold root | Four times the sampled root. |
| Adjusted root | Fourfold root minus 0x96. |
| Working base | Four times the adjusted root. |
| Anchor | Global word captured before the repeated-write region. |
| Late lookup result | Word obtained through the later byte-indexed lookup. |
| Fourfold loaded base | Four times the late lookup result. |
| Late retained base | Twice the late lookup result. |

Under 32-bit bitvector arithmetic, the small base is a multiple of four from zero through 508; its doubled value is a multiple of eight from zero through 1016. Low-address accesses are possible outcomes.

The adjustment subtracts 0x96 **after** multiplying the sampled root by four.

### Initial mutation and sampling

The high word of the final represented 64-bit result before direct mutation selects an offset-0x14 word destination. That destination receives the high-shift base. The high word’s low byte is then stored at high-shift-base offset nine.

An offset-0x54 word selected by the latest low result multiplied by 0x100 receives the independent scaled base. High-shift-base offset nine subsequently receives the low byte of the high-shift base, which is zero under 32-bit bitvector arithmetic. These are separate byte writes.

The independent scaled base’s offset-0x14 word is cleared before its offset-five byte is sampled. The doubled small base’s offset-0x14 word is then cleared, and the sampled byte is stored, zero-extended, at independent-scaled-base offset 0x34.

The independent scaled base’s cleared offset-0x14 word is read back. Another clear of doubled-small-base offset 0x14 follows. The read remains a live access: replacing it with zero requires ordinary-memory and noninterference assumptions not established here.

The doubled small base receives two byte writes at offset 0x15 using the read-back word’s low byte. The value associated with fixed location 0x05 is then read and widened for address selection. Fixed byte location 0x01 receives the doubled small base’s low byte.

Further effects include:

- A byte store at fixed-location-derived offset 0x15 using the small base.
- A retained call-derived byte written at independent-scaled-base offset 0x11, followed later by a clear there.
- A retained call-derived word written through the read-back word at offset 0x54.
- Capture of the old fixed-location-derived offset-0x54 word, followed by replacement with the read-back word.
- Publication of the small base at doubled-small-base offset 0x34.

The captured old word becomes the sampled root.

### Root publication

Subsequent effects include offset-0x74 pointer loads, writes through those loaded values, repeated self-valued stores at sampled-root offset 0x24, and duplicate byte stores. These relationships do not establish a linked structure.

A required ordering at doubled-small-base offset 0x14 and fixed word 0x54 is:

1. The calculated field receives the independent result shifted left by 26.
2. The 32-bit word beginning at fixed address 0x54 receives the sampled root shifted left by one.
3. The calculated field receives the unscaled sampled root twice.

The fixed-word payload is neither unspecified incidental state nor a derivative of the independent scaled base. Both shifts involve represented signed 32-bit quantities, with the arithmetic limitations already stated.

The fixed word can overlap calculated fields. Such overlap affects memory, but does not change the retained sampled-root value used for subsequent stores.

Later publication includes fourfold-root values, replacement of fourfold-root offset-0x54 and offset-0x14 fields, duplicate byte writes at doubled-small-base offset 0x15, a fixed byte write at 0x15, and offset-0x34 and offset-0x44 writes.

### Loaded-state relationships

Later addresses depend on:

- Offset-0x74 words selected through the doubled small base.
- A byte sampled at fourfold-root offset five and used to select an offset-0x74 word.
- A word loaded at fourfold-root offset 0x64.
- A word loaded at fourfold-root offset four, whose offset-five byte is sampled more than once.

The repeated byte samples have intervening mutations. Reusing an earlier sample is not established to preserve behavior.

Some loaded bases receive their own numerical value at offset 0x14. Offset-0x44 destinations receive, at different times, doubled loaded values, scaled-root values, or zero-extended byte samples. Byte destinations receive low portions of retained address-like words.

Every load observes memory at that access. Address expressions can overlap; stable fields and separate objects are not established.

### Width-specific constants and working state

The late lookup result selects three writes of numeric 0x3a:

| Destination | Width |
|---|---|
| Four times the lookup result | 8 bits |
| Eight times the lookup result | 16 bits |
| Four times the lookup result, plus 0x20 | 32 bits |

These widths are independently significant. No encoding or string layout is established.

Working-base offset 0x20 receives four 32-bit payloads in order:

**0x20 → 0x2f → zero-extended byte sampled at working-base offset four → 0x20.**

The byte sample occurs after the 0x2f write and before the corresponding word store.

The anchor is captured before the fixed 32-bit word at 0x60 receives 0x20. Later anchor-relative accesses use that retained capture.

### Repeated four-destination region

Each group has this ordered access pattern:

| Order | Destination | Payload |
|---|---|---|
| First | Scaled-address offset-0x54 word | Fourfold loaded base |
| Second | Anchor offset-0x19 byte | Working base’s low byte |
| Third | Working-base offset-0x11 byte | Anchor’s low byte |
| Fourth | Anchor offset-0x34 word | Group’s scaled address |

The first group uses eight times the sampled root as its scaled address. Subsequent groups use eight times the adjusted root.

This is fixed repetition, not a memory-controlled traversal. No fresh pointer traversal or payload sampling occurs between groups. Final assignments alone do not establish access-trace equivalence.

The numeric repetition count remains **documented_only** in this contract. No exact count is asserted, and this limitation does not authorize arbitrary write elimination.

### Fixed-byte mutation and interleaved captures

After the repeated region, the required ordering is:

1. Capture the 32-bit word beginning at fixed address 0xa9.
2. Capture the fourfold loaded base’s offset-0x74 word.
3. Write fixed byte location 0x3a.
4. Capture the late retained base’s offset-0x14 word.
5. Capture the working base’s offset-four word.

Under the documented low-byte arithmetic, the fixed-byte payload is the low byte of eight times the adjusted root plus 0x30. Substituting the root relationship gives 32 times the sampled root minus 0x480.

Consequently, its low five bits are zero. The possible byte values are:

**0x00, 0x20, 0x40, 0x60, 0x80, 0xa0, 0xc0, and 0xe0.**

A payload of 0x5a is not possible under this interpretation. Signed-character and overflow semantics remain subject to the stated uncertainty; this result describes the low-byte interpretation, not a resolution of every language-level arithmetic question.

The late retained base’s offset-0x14 capture can overlap fixed byte 0x3a and observe the write. The earlier two captures remain retained values.

The working-base capture also occurs after the byte write, but direct overlap is excluded under 32-bit bitvector address arithmetic: the working base is congruent to eight modulo 16, so its offset-four word starts at an address congruent to 0x0c and occupies low-nibble positions 0x0c through 0x0f. Fixed address 0x3a is outside that range. Ordering alone does not establish overlap.

### Final mutations and terminal invocation

The late retained base receives the working base’s low byte at offset 0x0d. Working-base byte fields receive 0x6c at offset eight and anchor-derived bytes at offsets 0x0d and nine.

Late-retained-base offset 0x54 first receives that base’s own value. Later it receives the retained word captured from offset 0x14 after the fixed-byte write.

Fixed byte 0x67 receives the low byte of the captured fixed-address word. Its current value supplies fixed byte 0x21 and later working-base offset-nine writes. Intervening calculated writes can alias 0x67, so these reads must not automatically reuse its initial payload.

A word loaded from late-retained-base offset 0x44 supplies:

- A byte at working-base offset 0x10.
- A word at anchor offset 0x54, subsequently replaced by the working base.
- A byte at late-retained-base offset one through a signed right shift.

That shift uses the working base’s low eight bits as its count. Negative operands and counts of 32 or greater remain **documented_only**.

A word is read at working-base offset 0x40. The retained working-base offset-four capture is published at anchor offset 0x74, then the offset-0x40 value is written at working-base offset 0x10, covering the earlier byte destination. Anchor offset 0x44 receives the late retained base.

Working-base offset 0x44 receives the captured fixed-address word and is later replaced by a word loaded at working-base offset 0x54. This loaded base receives a retained byte at offset 0x0d, a byte derived from a later working-base offset-0x14 read at offset 0x11, and decimal 100 at offset 0x54.

Near termination, a base loaded from late-retained-base offset four selects a 16-bit write of 0x0065 at twice that base. Its offset-0x64 and offset-0x74 words provide retained terminal arguments, with intervening writes between their captures.

A further word loaded from late-retained-base offset 0x64 selects an offset-0x54 destination receiving the terminal second argument. A word at twice that argument’s value receives the further loaded word.

The terminal invocation has these represented arguments:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | Offset-0x64 word from the base loaded at late-retained-base offset four |
| Third | Retained late-retained-base offset-0x14 capture |
| Fourth | Later offset-0x74 word from the same base used for the second argument |

The third argument is not reloaded. The second and fourth arguments are not an atomic snapshot. A return without a value follows if the terminal invocation returns.

## Side effects

Direct accesses use 8-, 16-, and 32-bit widths. Offsets describe byte-address relationships, not established structure layouts.

Unsigned byte samples widened to words are zero-extended. Established low-byte stores retain eight bits. Memory byte order, alignment tolerance, atomicity, concurrency, and special-memory behavior are unresolved.

Fixed-location effects include byte writes at 0x01, 0x15, 0x21, 0x3a, and 0x67; word writes at 0x54 and 0x60; and a word read at 0xa9. The independently declared width of the value associated with 0x05 is unavailable.

Calculated accesses may overlap fixed locations or partially overlap wider accesses where arithmetic permits. Retained values do not change retroactively when their former storage is overwritten.

No direct allocation, release, ownership transfer, locking, rollback, transaction, or explicit error-reporting mechanism is established. Dependency ownership and memory effects must not be assumed. Failure may follow partial mutation.

## Dependencies

The supplied identity mapping establishes only F-3f2afbaf9262. No opaque function IDs are supplied for the indirect dependencies; their identities remain **documented_only**. No substitute identities are invented.

Dependency boundaries comprise the opening invocation, ordered intermediate invocations, and terminal invocation. No dependency is established to terminate, return normally, preserve globals or memory, or preserve ownership. Similar target constructions do not prove target identity.

## Acceptance examples

These are conditional static examples, not executed tests.

### Input forwarding

For explicit inputs 0x12345678 and 0x9abcdef0 and first incoming state word 0x00001800, the opening arguments are those two inputs followed by 3.

### Extraction and low-address boundary

For 0xffabcdef, extracting the low 24 bits and shifting right by eight yields 0xabcd; shifting the low 24 bits right by 19 yields 0x15.

A small-base source result of 0x7f produces small base 508 and doubled small base 1016 under bitvector arithmetic. Access validity remains an environmental requirement.

### Ordered fixed-word publication

Assume defined calculations, independent result 1, and sampled root 0x100. The relevant payload sequence is 0x04000000 at doubled-small-base offset 0x14, then 0x00000200 at fixed word 0x54, then two payloads of 0x100 at the calculated field.

If that field begins at 0x54, these are successive writes to the same word. Its final value does not demonstrate the intermediate effects.

### Reachable fixed-byte payload and subsequent capture

Assume sampled root 0x26. The fourfold root is 0x98, the adjusted root is 2, and the working base is 8. Eight times the adjusted root plus 0x30 is **0x40**, a reachable fixed-byte payload.

Further assume little-endian ordinary memory and late lookup result 0x12. The late retained base is then 0x24, and its offset-0x14 word begins at 0x38.

If that word contains 0x11223344 immediately before the fixed-byte write, storing 0x40 at 0x3a makes the subsequent capture **0x11403344**. The terminal third argument retains that captured word despite later overwrites of its storage.

For this root, the working-base offset-four word begins at 0x0c and does not overlap 0x3a. Different byte order changes the affected bit positions in the overlapping late-retained-base word.

### Live reads and partial progress

If an intervening calculated write changes byte 0x67, a subsequent read there uses its current value, not necessarily the initially published byte.

If an unsupported access occurs after successful stores, no rollback or clean failure status is established.

## Uncertainties

- **documented_only — Execution:** complete entry state, implicit results, calling convention, reachability, and terminal continuation.
- **documented_only — Dependencies:** identities, signatures, global effects, memory effects, and ownership behavior.
- **documented_only — Arithmetic:** signed overflow, negative shifts, oversized shift counts, character signedness, and address wrapping.
- **documented_only — Memory:** byte order, alignment, atomicity, concurrency, special-memory behavior, and failure visibility.
- **documented_only — Coverage:** numeric repetition count and complete intermediate invocation signatures.
- **documented_only — Domain meaning:** schemas, encodings, resource protocols, and application invariants.
- **Inferred intent:** related repeated stores resemble publication, but synchronization and an external observer are not established.

## Evidence

The supplied body supports the represented widths, retained-value dependencies, fixed-location effects, arithmetic relationships, and relative access ordering described here.

The fixed word at 0x54 is written between the independent-result-derived and sampled-root-derived stores. The fixed byte at 0x3a is written after the fixed-address and offset-0x74 captures, but before the late offset-0x14 and working-base offset-four captures.

The reachable-payload constraint follows from composing the supplied root scaling, subtraction, and byte reduction. The exclusion of direct working-base-offset-four overlap follows separately from its address congruence under bitvector arithmetic.

Comments, prior prose, and feedback are not independent behavioral evidence. No runtime tests, dependency inspection, or instruction-level corroboration have been performed. Conditional examples do not resolve the documented machine-semantics uncertainties.
