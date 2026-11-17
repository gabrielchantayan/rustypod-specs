<!-- Trusted provenance: F-5ac86bda2c5d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ac86bda2c5d

## Purpose

F-5ac86bda2c5d invokes a long sequence of unresolved indirect dependencies, transforms their results and implicit incoming state, performs alias-sensitive mixed-width memory updates, and makes a final indirect invocation.

The supplied static representation does not establish an application-level purpose. Constants resembling character values do not establish text generation, serialization, or stream output.

This contract distinguishes:

- **Established:** behavior explicitly represented by the supplied static material.
- **[INFERENCE]:** conclusions conditional on additional machine-model assumptions.
- **documented_only:** behavior not resolved here.

Every effect is conditional on execution reaching it. A dependency may not return, and a memory access may fault.

## Interface

### Inputs and completion

The represented interface declares no explicit parameters and no scalar return value. Effective inputs nevertheless include two incoming 32-bit machine-state values, an incoming control-location value, primary and secondary dependency results, calculated memory contents, fixed low-address storage, a separately loaded global base, and an unresolved final target.

A parameterless declaration does not establish input independence or a portable calling convention.

Observable outputs comprise dependency invocations, represented arguments, ordered memory accesses, and the final invocation. That invocation has four represented arguments: `0x72`, a late-loaded word, an earlier retained snapshot, and another late-loaded word. Their provenance appears below.

A return is represented after the final invocation. Successful completion requires that target to return; no explicit scalar result is supplied by F-5ac86bda2c5d.

### Widths and arithmetic

| Quantity | Represented width |
|---|---:|
| Byte accesses | 8 bits |
| Halfword stores | 16 bits |
| Word accesses and most arithmetic | 32 bits |
| Selected dependency results | 64 bits |

Explicit unsigned byte loads are zero-extended when used as words. Conventional narrowing retains the low eight or sixteen bits for byte or halfword stores.

Numeric extraction of a 64-bit result’s lower and upper 32-bit words does not establish memory endianness.

**[INFERENCE]:** Under a 32-bit bitvector model, address calculations, multiplication, and left shifts retain their low 32 bits. Signed-looking expressions prevent treating that model as an unconditional portable arithmetic guarantee. Signed overflow, exceptional narrowing, signed right shifts, and excessive shift counts require actual machine semantics.

The symbol-associated input at fixed address `0x05` is widened to a word. Its declaration is absent; its exact read width and signedness remain **documented_only**.

### Preconditions

Traversal requires executable indirect targets, compatible calling conventions, usable implicit result channels, and permitted memory accesses of every reached width and alignment. Fixed low addresses must be meaningful in the environment.

No explicit null, bounds, alignment, or error-status checks are represented.

## Behavior

### Indirect dependency phase

The opening phase is straight-line: no conditional branch or loop is represented. The first indirect target derives from incoming control state plus `0xffe`. Subsequent targets predominantly derive from embedded control-location values or code-associated references plus the same offset. Similar construction does not establish a common dependency identity.

Opening argument relationships include the first incoming word shifted left by 24; a returned primary word shifted left by 15 and supplied twice; and another primary word shifted left by eight paired with the second incoming word shifted left by three. A subsequent invocation receives a 64-bit result’s lower word shifted left by eight, its upper word, the transformed second incoming word, and a zero-derived argument.

Later invocations consume retained results across intervening calls, secondary result channels, duplicate arguments, zeros, masked fields, and shifted values. Calls without displayed arguments still have unknown implicit inputs and effects.

Transformations include low-field masks of widths between 11 and 24 bits, left shifts up to 31 positions, unsigned right shifts, and full-width 64-bit right shifts by 34, 36, or 44 before narrowing. Those full-width shifts numerically correspond to the upper word shifted right by two, four, or twelve.

Under the bitvector model, masking to 20 bits and shifting right by ten yields at most 1023; masking to 24 bits and shifting right by 20 yields at most 15. Transformations that discard all original information do not eliminate the observable invocations receiving their results.

The complete invocation-by-invocation argument schedule remains **documented_only** in this contract. No dependency is assumed pure, globally preserving, or normally returning.

### Value and address vocabulary

These descriptive terms express provenance, not recovered object types:

| Term | Meaning |
|---|---|
| Large base | A retained primary result multiplied by `0x02000000` |
| Narrow base | A different retained result shifted left by 25 |
| Small value | Another result shifted left by eight, masked to 15 bits, then shifted unsigned right by six |
| Twice-small base | Twice the small value |
| Shifted payload | A retained primary result shifted left by eight |
| Early retained word | A later word read at large-base displacement `0x14` |
| Saved word | A word read through the fixed-input-derived base |
| Fourfold saved base | Four times the saved word |
| Adjusted base | Fourfold saved base minus `0x96` |
| Scaled adjusted base | Four times the adjusted base |

Under bitvector semantics, the small value is a multiple of four between zero and 508; the twice-small base is a multiple of eight through 1016. No non-aliasing, allocation, ownership, or record layout follows from these terms.

### Initial direct effects

Direct memory effects begin after the final represented 64-bit dependency result. Its upper word supplies the first destination base; its lower word supplies another scaled destination.

A word at upper-word displacement `0x14` receives the narrow base. Narrow-base displacement nine then receives the upper word’s low byte. Before that byte destination receives the narrow base’s low byte, a word at final-lower-word-times-256 displacement `0x54` receives the large base.

Large-base displacement `0x14` is zeroed before an unsigned byte is read at displacement five. Twice-small displacement `0x14` is then zeroed, and large-base displacement `0x34` receives the zero-extended byte.

The subsequent word read at large-base displacement `0x14` supplies the early retained word. It cannot unconditionally be replaced by zero: intervening stores may overlap it.

Twice-small displacement `0x14` is zeroed again, followed by two byte stores of the early retained word’s low byte at twice-small displacement `0x15`.

### Fixed input and saved-word propagation

The input at `0x05` is read before fixed byte address `0x01` receives the narrowed twice-small base. The widened input becomes the fixed-input-derived base.

Before the saved-word load, fixed-input-derived displacement `0x15` receives the small value’s low byte. Large-base displacement `0x11` receives the shifted payload’s low byte; early-retained-word displacement `0x54` receives the full shifted payload; then large-base displacement `0x11` is overwritten with zero.

The saved word is loaded from fixed-input-derived displacement `0x54` before that location receives the early retained word. Later computations use the saved snapshot, not a reread.

Twice-small displacement `0x34` receives the small value. A word read at large-base displacement `0x74` supplies a retained destination base. Its displacement `0x54` first receives the primary result underlying the large base shifted left by 26.

A separate self-referential update uses two represented word reads at twice-small displacement `0x74`: one supplies a destination base and one supplies a stored word at destination displacement `0x54`. Their relative evaluation order is unresolved; equal results are not guaranteed.

The saved word is stored twice at saved-word displacement `0x24`. Its low byte is stored twice at displacement nine of the retained destination base, with a shifted-payload word store at that base’s displacement `0x54` between the two pairs.

The following ordering is established and must remain explicit because destinations can overlap:

| Relative order | Word-store destination | Value |
|---|---|---|
| First | Twice-small displacement `0x14` | Primary result underlying the large base shifted left by 26 |
| Second | Fixed address `0x54` | Saved word shifted left once |
| Third | Twice-small displacement `0x14` | Saved word |
| Fourth | Twice-small displacement `0x14` | Saved word again |

The fixed-address store precedes **both** saved-word stores; it is not between them.

After those stores, two byte stores at twice-small displacement `0x15` contain the fourfold saved base’s low byte. Only afterward does fixed byte address `0x15` receive the narrowed twice-small base—not the fourfold saved base or original small value.

### Further mixed-width propagation

Fourfold-saved displacement `0x54` receives the shifted payload and subsequently the fourfold saved base. Twice-small displacement `0x34` also receives the fourfold value.

A word loaded at twice-small displacement `0x74` is doubled. This **doubled link** is stored at saved-word displacement `0x44`.

An unsigned byte read at fourfold-saved displacement five selects a word read at byte-value displacement `0x74`. This **byte-selected link** receives the doubled link at displacement `0x44`. Its low byte propagates to displacement-`0x0d` byte destinations, while corresponding calculated bases receive their own word values at displacement `0x14`.

A word read at fourfold-saved displacement `0x64` supplies another destination for that byte-and-self-word update. Another word read, at fourfold-saved displacement four, supplies a base for two unsigned byte reads at displacement five.

Between those byte reads, stores update the doubled link and fourfold saved base. The first byte supplies a zero-extended word store at byte-selected-link displacement `0x44`; the second selects another displacement-`0x74` word read. That result is the **selection word**; twice it is the **tail base**.

The doubled link receives low-byte updates and displacement-`0x44` stores containing eightfold or fourfold saved-word values. The fourfold saved base receives additional displacement-nine, `0x14`, `0x15`, and `0x54` updates.

The full interleaving of this mixed region remains **documented_only** here. Established snapshots and expressly specified ordering are not included in that limitation.

### Constants and repeated writes

The selection word supplies three destinations receiving `0x3a`: a byte through four times the selection word, a halfword through eight times it, and a word at four-times-selection displacement `0x20`.

Scaled-adjusted displacement `0x20` then receives four successive word stores: `0x20`, `0x2f`, a zero-extended byte read from scaled-adjusted displacement four, and `0x20`. The byte read occurs after the `0x2f` store.

The global base is loaded next, before fixed word address `0x60` receives `0x20` and before the repeated region.

Each repeated group preserves four ordered stores:

| Order | Effect |
|---|---|
| First | Word at an eight-times-scaled base plus `0x54` receives four times the selection word |
| Second | Byte at global-base displacement `0x19` receives the scaled adjusted base’s low byte |
| Third | Byte at scaled-adjusted displacement `0x11` receives the global base’s low byte |
| Fourth | Word at global-base displacement `0x34` receives that group’s eight-times-scaled base |

The first group uses eight times the saved word; subsequent groups use eight times the adjusted base. The global base and narrowed values are retained rather than visibly reloaded per group. Exact repetition multiplicity remains **documented_only** in this contract. Repeated effects must not be collapsed merely because values and destinations recur.

### Post-repetition reads and snapshots

The established access order immediately after repetition is:

1. Word read at fixed address `0xa9`.
2. Word read at four-times-selection displacement `0x74`.
3. Byte write at fixed address `0x3a`.
4. Word read at tail-base displacement `0x14`.
5. Word read at scaled-adjusted displacement four.

The `0x3a` byte derives from narrowing the adjusted base to a character-like eight-bit quantity, multiplying by eight, adding `0x30`, and narrowing again. Under conventional promotion and modulo-256 final narrowing, its value is the low eight bits of eight times the adjusted base plus `0x30`.

The two following word reads must not move before that byte store without an aliasing argument. Their results are independently retained snapshots.

Before the next fixed-byte update, tail-base displacement `0x0d` receives the scaled adjusted base’s low byte. Scaled-adjusted displacement eight receives `0x6c`; displacements `0x0d` and nine receive the global base’s low byte. Tail-base displacement `0x54` receives the tail base.

### Fixed-byte rereads and restoration

Fixed byte address `0x67` receives the low byte of the earlier `0xa9` word. A word is then read at tail-base displacement `0x44`; only afterward is `0x67` read to supply fixed byte address `0x21`.

The calculated word supplies a byte at scaled-adjusted displacement `0x10`, a word at global-base displacement `0x54`, and a shifted byte at tail-base displacement one. The shift is signed-looking, with count equal to the scaled adjusted base’s low eight bits. No five-bit count mask is represented; behavior for counts of 32 or greater remains **documented_only**.

Global-base displacement `0x54` is subsequently overwritten with the scaled adjusted base. A word at scaled-adjusted displacement `0x40` is retained while global-base displacement `0x74` receives the earlier displacement-four snapshot. The retained `0x40` word is then stored at scaled-adjusted displacement `0x10`; global-base displacement `0x44` receives the tail base.

Two later reads of fixed byte `0x67` independently supply byte stores at scaled-adjusted displacement nine. Between those reads, calculated stores update tail-base displacement nine and scaled-adjusted displacement `0x44`, followed by a word read at scaled-adjusted displacement `0x54`.

Neither reread may unconditionally use the original retained low byte from `0xa9`: intervening stores may alias `0x67`.

The newly loaded displacement-`0x54` word supplies a destination base. Its displacement `0x0d` receives the low byte of the post-repetition four-times-selection displacement-`0x74` snapshot.

Only then is the retained tail-base displacement-`0x14` snapshot restored at tail-base displacement `0x54`. Restoration does not reread displacement `0x14`.

### Final dependent updates and invocation

After restoration, a word is read at scaled-adjusted displacement `0x14`; its low byte is retained. Tail-base displacement `0x11` receives the earlier post-repetition displacement-`0x74` snapshot’s low byte. Scaled-adjusted displacements nine and `0x44` receive the newly loaded destination base’s low byte and full word.

A word read at tail-base displacement four supplies the **final source base**. A halfword store of `0x65` through twice that base precedes the word read at final-source displacement `0x64`.

The previously loaded destination base receives the retained scaled-adjusted displacement-`0x14` byte at displacement `0x11`, then word value 100 at displacement `0x54`.

Final-source displacement `0x74` is read before tail-base displacement `0x64`. Tail-base displacement nine then receives the final source base’s low byte. The tail-base `0x64` result supplies a destination whose displacement `0x54` receives the saved final-source `0x64` word. A word through twice that saved source word receives the tail-base `0x64` result.

The final invocation receives, in order:

- `0x72`.
- The saved final-source displacement-`0x64` word.
- The post-repetition tail-base displacement-`0x14` snapshot.
- The final-source displacement-`0x74` word.

## Side effects

Mixed-width accesses may overlap calculated destinations, fixed storage, or global-base storage. Displacements describe accesses, not object layouts. Numeric effects of partial overlap depend on endianness.

The word read at `0xa9` is not naturally four-byte aligned. Alignment support, fault behavior, and specialized memory semantics are unknown.

No allocation, release, ownership transfer, or lifetime guarantee is established. Self-stores do not prove a particular data structure. Unknown dependencies may affect memory or ownership; this contract assigns them no preservation or release guarantees.

No locking, atomic transaction, rollback, or explicit recovery path is represented. Later failure does not visibly undo earlier effects. Concurrency, reentrancy, fault propagation, and nonlocal exits remain **documented_only**.

## Dependencies

F-5ac86bda2c5d is the only supplied opaque function ID. No opaque IDs are supplied for the indirect dependencies or final target; their identities remain unresolved rather than being invented.

Calling conventions, secondary result behavior, global preservation, ownership effects, release behavior, and normal return are unknown. An invocation with no displayed arguments is not assumed effect-free.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

1. **Opening width:** Under 32-bit bitvector semantics, an incoming word `0x00000123` produces first argument `0x23000000`.

2. **Result splitting:** A 64-bit result `0x1122334455667788` has lower word `0x55667788` and upper word `0x11223344`, independently of memory endianness.

3. **Small-value boundary:** A contributing result with low seven bits `0x7f` produces small value 508 and twice-small base 1016. The later fixed byte at `0x15` receives `0xf8` under conventional narrowing.

4. **Saved snapshot:** A saved-word load returning `0x12345678` remains that snapshot after its source location is overwritten. The fixed-address `0x54` shifted-word store contains `0x2468acf0` under bitvector semantics.

5. **Overlapping corrected sequence:** If the small value is 32, twice-small displacement `0x14` equals fixed address `0x54`. That address receives the left-by-26 value, then the shifted saved word, then the saved word twice. Immediately after the second saved-word store, before subsequent byte stores, its word value is the saved word. Placing the fixed-address shifted store between the saved-word stores is not acceptable, even if a final-memory-only comparison misses the difference.

6. **Post-repetition fixed write:** An adjusted base of `0x122` produces byte `0x40` at `0x3a` under conventional narrowing. Any overlapping retained-word read specified afterward observes memory after that byte write.

7. **Fresh reads and restoration:** An intervening store changing `0x67` can affect a later represented reread. In contrast, restoration to tail-base displacement `0x54` uses the retained displacement-`0x14` snapshot despite intervening changes to its original source.

8. **Partial completion:** A later fault requires neither successful return nor rollback of completed effects.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object schemas, ownership, and lifetimes.
- Complete dependency argument schedule, dependency identities, and implicit calling convention.
- Exact repeated-store multiplicity and the unenumerated mixed-region interleaving.
- Relative evaluation order of the two reads within the self-referential twice-small-link update.
- Memory map, alias relationships, endianness, alignment, and concurrency semantics.
- Exact read width and signedness of the input associated with `0x05`.
- Signed overflow, exceptional narrowing, signed right shifts, and excessive shift counts.
- Validity of the represented function boundary and actual final-transfer behavior.

These limitations do not permit removing, merging, or reordering established accesses. In particular, the fixed word store at `0x54` is established to follow the left-by-26 twice-small store and precede both saved-word stores.

## Evidence

The supplied static representation is the sole behavioral evidence. Prior prose, reviewer feedback, embedded comments, and warnings are not independent proof.

Established observations include implicit inputs, indirect invocations, numeric result splitting, explicit masks and shifts, mixed-width accesses, saved snapshots, fixed-address sequencing, repeated writes, restoration, and final argument provenance.

The saved-word propagation region independently establishes the corrected ordering: the left-by-26 destination update precedes the fixed-address shifted-word update, which precedes both saved-word updates.

Bitvector arithmetic and narrowing examples are explicitly conditional. No dependency implementation, executable behavior, or memory map has been validated. No runtime tests have been performed.
