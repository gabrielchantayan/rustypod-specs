<!-- Trusted provenance: F-c2a443dda60f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c2a443dda60f

## Purpose

F-c2a443dda60f performs an ordered sequence of indirect calls, integer transformations, and memory mutations, followed by an unresolved indirect transfer. No application-level purpose is established.

This contract separates:

- **Established static evidence:** represented operations, explicit pointer-access widths, destination-local widths, captured dataflow, and statement ordering.
- **Inferred intent:** possible interpretations of address-bearing values and field-shaped accesses.
- **documented_only:** unresolved behavior that cannot support an executable guarantee.

Character-valued constants do not establish text output. Self-reference-shaped stores do not establish a linked structure. No allocation, ownership transfer, or successful completion is implied.

## Interface

### Entry and completion

The apparent interface has no explicit parameters or declared result. Its implicit inputs nevertheless include:

- Two incoming 32-bit values used in shifted arguments.
- An incoming control-related value used to derive the first indirect target.
- Dependency results, including represented 64-bit paired results.
- Additional post-call words whose relationship to ordinary results is unresolved.
- Memory contents obtained through derived addresses and undeclared global symbols.
- An unresolved final transfer target.

The actual calling convention and safe entry procedure are **documented_only**. An ordinary no-argument invocation is not established as valid.

There is no explicit success or error result. A return is represented after the final transfer, but whether that transfer returns or redirects execution permanently is **documented_only**.

### Widths and conversions

| Operation or quantity | Established property |
|---|---|
| Explicit one-byte pointer access | 8-bit memory access |
| Explicit unsigned-byte load | Result between 0 and 255 |
| Explicit halfword pointer store | 16-bit memory access |
| Explicit word pointer access | 32-bit memory access |
| Ordinary represented integer | 32 bits |
| Paired result | 64 bits, with separately consumed 32-bit portions |
| Global at 0x05 captured into unsigned storage | Destination is unsigned 32-bit; source access width and source-dependent conversion are unresolved |
| Anchor and 0xa9 global captures | Destination locals are 32-bit; source access widths and source-dependent conversions are unresolved |
| Assignments to undeclared globals | Right-hand conversions may be explicit; destination access width and destination-type conversion remain unresolved |

All offsets are byte offsets. Multi-byte byte order is unresolved. Explicit narrowing to one-byte storage establishes an eight-bit destination representation; signed character conversion rules are not independently established.

**No declaration is supplied for any global symbol used here.** This limitation applies to the symbols associated with 0x01, 0x05, 0x15, 0x21, 0x3a, 0x54, 0x60, 0x67, 0xa9, and 0x087fbbbc.

Consequently:

- A global expression captured into a 32-bit local does not establish a 32-bit source access.
- An explicitly one-byte-narrowed value assigned to a global does not establish a one-byte destination access.
- A global expression stored through a one-byte pointer does not establish a one-byte global read.
- Global-to-global assignment does not establish equal source and destination types or access widths.

These globals’ access widths, signedness, and declaration-dependent conversions are **documented_only**.

Unsigned right shifts are logical. Signed right shifts, signed overflow, and signed left-shift validity require execution-model rules not supplied here. Relationships stated under the **32-bit bit-vector model** retain the low 32 bits of intermediate results; that model does not establish valid pointers.

### Preconditions and failures

Every reached indirect target must be callable under the actual convention. Every reached memory access must support its direction and represented width; accesses to undeclared globals additionally require their missing declarations.

No explicit null, alignment, bounds, or shift-count checks are represented. Low addresses and small derived addresses are used directly. Valid mappings, ordinary-RAM semantics, object lifetimes, and disjointness are not established.

Invalid targets, accesses, arithmetic, or shifts have no represented recovery path.

## Behavior

### Dependency phase

A long, straight-line sequence of indirect calls precedes the direct mutation phase. No explicit loop or conditional branch appears in that sequence. Later operations are reached only if earlier calls return.

Established argument relationships include:

- The first call receives an incoming word shifted left by 24.
- Some returned words are supplied twice, with each copy shifted left by 15.
- The other incoming word contributes arguments shifted left by 3, 9, and 10.
- Returned low words frequently contribute left-eight-shifted arguments.
- Paired results contribute high words independently of transformed low words.
- Some calls receive explicit zeros or equal arguments.
- Captured values remain inputs across intervening calls.
- Additional post-call words contribute separately from ordinary results.

No-argument calls may consume implicit state. Discarded results do not make calls removable. Similar argument patterns do not establish a common target.

Established extraction relationships include:

| Transformation | Result |
|---|---|
| Low-20-bit mask, then logical right shift by 10 | Bits 10–19; range 0–1023 |
| Low-24-bit mask, then logical right shift by 20 | Range 0–15 |
| Low-24-bit mask, then logical right shift by eight | Range 0–65535 |
| Low-17-bit mask, then logical right shift by four | Range 0–8191 |
| Unsigned 64-bit right shifts by 34, 36, or 44 | High word logically shifted by 2, 4, or 12 |

Under the bit-vector model, left eight followed by logical right eight discards the original high byte. Explicitly zeroed scalar values remain zero through subsequent scalar transformations; this establishes nothing about dependency side effects.

### Address-bearing roles

These descriptive roles identify captured dataflow, not proven objects.

| Role | Derivation |
|---|---|
| Primary source | Ordinary dependency result captured near the end of the call phase |
| Primary base | Primary source multiplied by \(2^{25}\) |
| Packed base | Earlier paired result’s low word shifted left by 25 |
| Small index | Later result shifted left by eight, masked to 15 bits, then logically shifted right by six |
| Small base | Twice the small index |
| Captured-global base | Unsigned 32-bit conversion of the global expression associated with 0x05 |
| Loaded base | Explicit word load at captured-global offset 0x54 |
| Scaled base | Four times the loaded base |
| Adjusted base | Scaled base minus 150 |
| Working base | Four times the adjusted base |
| Anchor | 32-bit local capture from the global associated with 0x087fbbbc |
| Terminal seed | Word obtained through an explicitly unsigned-byte-derived address |
| Terminal base | Twice the terminal seed |
| Doubled-terminal address | Four times the terminal seed |

Under the bit-vector model, the small index is a multiple of four between 0 and 508, the small base is a multiple of eight between 0 and 1016, and the working base is congruent to eight modulo sixteen.

Neither the captured-global base nor the anchor has a source-width-derived eight-bit bound.

### Opening mutations and global capture

The last paired result supplies the opening destinations. Its high word supplies an offset-0x14 word destination receiving the packed base. Packed offset 9 receives a character-converted high-word value. A word store of the primary base at the last low word multiplied by 256 plus 0x54 intervenes before another packed-offset-9 store.

Primary offset 0x14 is cleared before an explicit unsigned-byte load at primary offset 5. Small offset 0x14 is cleared, primary offset 0x34 receives the widened byte, and primary offset 0x14 is read back. Aliasing means this captured word need not remain zero.

After another small-offset-0x14 clear, small offset 0x15 receives the read-back word’s one-byte conversion twice.

The global expression associated with 0x05 is then read and converted into the unsigned 32-bit captured-global base. **This capture precedes the assignment to the global associated with 0x01.** That assignment supplies an explicitly one-byte-narrowed small-base value; its global destination width is unknown.

Subsequent ordered effects are:

- Captured-global offset 0x15 receives a one-byte conversion of the small index.
- Primary offset 0x11 receives a one-byte conversion of an earlier left-eight-shifted argument.
- The primary read-back word supplies a base whose offset 0x54 receives that earlier argument as a word.
- Primary offset 0x11 is cleared.
- Captured-global offset 0x54 is read to capture the loaded base, then overwritten with the primary read-back word.
- Small offset 0x34 receives the small index as a word.

Later changes to the 0x05 global do not recapture the base. Captured-global offset 0x54 is not the global associated with fixed address 0x54 unless the actual addresses coincide.

### Intermediate mutations and fixed-global assignments

A word from primary offset 0x74 supplies another base. Its offset 0x54 first receives the primary source shifted left by 26.

A separate self-reference-shaped update uses two represented word reads at small offset 0x74, one for the destination base and one for the stored value. Their relative operand-evaluation order is not independently established.

Loaded offset 0x24 receives the loaded base twice. Between these stores, the primary-offset-0x74-derived base receives an offset-9 byte and an offset-0x54 word. Its offset-9 byte is written again afterward.

The following statement ordering is established:

1. Small offset 0x14 receives the primary source shifted left by 26.
2. The global associated with 0x54 is assigned the loaded base shifted left by one.
3. Small offset 0x14 receives the loaded base twice.
4. Small offset 0x15 receives a one-byte conversion of the scaled base twice.
5. The global associated with 0x15 is assigned an explicitly one-byte-narrowed small-base value.

The two global destination widths are **documented_only**. Neither assignment may be described as a proven word or byte memory access.

Further explicit mutations involve scaled offsets 9, 0x0d, 0x14, 0x15, and 0x54, and loaded offset 0x44. Important captures include:

- A fresh word at small offset 0x74 supplies a doubled address-bearing value.
- An unsigned byte at scaled offset 5 supplies the base for an offset-0x74 word load.
- Scaled offset 0x64 supplies another base receiving offset-0x0d and offset-0x14 updates.
- A word at scaled offset 4 supplies a base whose offset-5 unsigned byte is loaded twice.
- Stores intervene between those two byte loads; their observations need not agree.
- The second observation supplies the base for the offset-0x74 word load capturing the terminal seed.

These explicit unsigned-byte loads, unlike undeclared-global expressions, establish values between 0 and 255.

### Constants, working state, and anchor

The terminal seed supplies destinations receiving 0x3a through three explicit widths: a byte at four times the seed, a halfword at eight times the seed, and a word at four times the seed plus 0x20.

Working offset 0x20 receives four word writes, in order: 0x20, 0x2f, the zero-extended byte loaded at working offset 4, and 0x20 again. The byte load occurs after the second write and before the third.

**The anchor global is read after the fourth write and before the assignment of 0x20 to the global associated with 0x60.** The anchor capture has a 32-bit destination, not a proven 32-bit source access. The 0x60 assignment has no established global destination width.

Next, eight times the loaded base plus 0x54 receives the doubled-terminal address as a word.

A finite repeated block writes a one-byte working-base conversion at anchor offset 0x19, a one-byte anchor conversion at working offset 0x11, and a word at anchor offset 0x34. The first word is eight times the loaded base; later groups use eight times the adjusted base. Between groups, eight times the adjusted base plus 0x54 receives the doubled-terminal address.

The numeric repetition cardinality remains **documented_only** here. This does not permit arbitrary repetition counts or collapsing represented accesses. The captured anchor is reused without another represented read of its source global.

### Final captures and separately timed global reads

The global associated with 0xa9 is read into a 32-bit local capture. Its source access width and conversion remain unresolved. A word at doubled-terminal address plus 0x74 is captured next.

Only afterward is the global associated with 0x3a assigned an arithmetic expression consisting of the character-converted adjusted base, multiplied by eight, plus 0x30. **There is no explicit final one-byte narrowing of this expression.** Its global destination width and declaration-dependent conversion are unknown. Describing this assignment merely as storing the expression’s low byte would lose established arithmetic.

Terminal offset 0x14 and working offset 4 are then captured as words. Before the assignment to the 0x67 global:

- Terminal offset 0x0d receives a one-byte working-base conversion.
- Working offset 8 receives byte 0x6c.
- Working offsets 0x0d and 9 receive a one-byte anchor conversion.
- Terminal offset 0x54 receives the terminal base as a word.

The 0x67 assignment supplies an explicitly one-byte-narrowed value from the **saved 32-bit 0xa9 capture**, not a new 0xa9 read. Its destination access width remains unknown.

Terminal offset 0x44 is read next. The 0x67 global is then read afresh and assigned to the 0x21 global. Neither global access width nor the intervening type-dependent assignment conversion is established.

The terminal-offset-0x44 capture supplies a byte at working offset 0x10 and a word at anchor offset 0x54. A signed-right-shift-derived character value is stored at terminal offset 1; the count is the working base’s low eight bits. Anchor offset 0x54 subsequently receives the working base.

Working offset 0x40 is captured before anchor offset 0x74 receives the saved working-offset-4 word. Working offset 0x10 then receives the offset-0x40 capture as a word, overlapping its earlier byte store. Anchor offset 0x44 receives the terminal base.

Three later working-offset-9 stores have distinct sources:

| Store | Source and capture timing |
|---|---|
| First | Fresh 0x67 global expression after the anchor-offset-0x44 store |
| Second | Another fresh 0x67 expression after terminal offset 9 is written, working offset 0x44 receives the saved 0xa9 capture, and working offset 0x54 is read |
| Third | One-byte conversion of the saved working-offset-0x54 word, after further stores and a working-offset-0x14 capture |

The first two destinations are explicitly one byte wide; their global source reads are not proven byte reads. Including the read supplying the 0x21 assignment, there are **three separately timed reads of 0x67 after its assignment**. Intervening aliases may affect their observations.

### Terminal updates and transfer

The saved doubled-terminal-offset-0x74 word supplies one-byte stores at offset 0x0d of the working-offset-0x54-derived base and at terminal offset 0x11. Terminal offset 0x54 is restored from the saved terminal-offset-0x14 word. Working offset 0x44 receives the saved working-offset-0x54 word.

A word at terminal offset 4 supplies a halfword destination at twice that word, receiving 0x65, and a base for offset-0x64 and later offset-0x74 word captures.

Between those captures, the working-offset-0x54-derived base receives a one-byte conversion of the saved working-offset-0x14 word at offset 0x11 and word value 100 at offset 0x54.

A later word at terminal offset 0x64 supplies another base. Terminal offset 9 receives a character conversion of the terminal-offset-4 capture. The later base’s offset 0x54 receives the saved offset-0x64 word, and a word destination at twice that saved word receives the later base.

The final unresolved transfer receives:

1. 0x72.
2. The saved offset-0x64 word obtained through the terminal-offset-4-derived base.
3. The saved terminal-offset-0x14 word.
4. The later offset-0x74 word obtained through the terminal-offset-4-derived base.

No success check or recovery follows.

## Side effects

Explicit pointer operations establish repeated and overlapping 8-, 16-, and 32-bit accesses. For example, offset 0x15 overlaps a word beginning at offset 0x14, and working offset 0x10 receives both byte and word stores.

Global effects must be distinguished from these explicitly sized accesses:

| Global-associated address | Established expression-level effect |
|---|---|
| 0x05 | Read converted into unsigned 32-bit capture |
| 0x01 | Assignment from explicitly one-byte-narrowed small base |
| 0x54 | Assignment from loaded base shifted left by one |
| 0x15 | Assignment from explicitly one-byte-narrowed small base |
| 0x087fbbbc | Read captured into 32-bit anchor |
| 0x60 | Assignment of 0x20 |
| 0xa9 | Read captured into 32-bit storage |
| 0x3a | Assignment of character-converted adjusted base times eight plus 0x30 |
| 0x67 | Assignment from narrowed saved capture, followed by three separately timed reads |
| 0x21 | Assignment from the first subsequent 0x67 read |

**Every global access width and declaration-dependent conversion in this table is documented_only.** Derived accesses intervene; rows do not imply consecutive operations or non-overlap.

Unknown dependencies may mutate globals, invalidate memory, perform I/O, or alter control flow. No preservation, allocation, release, ownership-transfer, thread-safety, atomicity, rollback, or idempotence guarantee is established.

## Dependencies

| Opaque ID | Role |
|---|---|
| F-dep-001 | First indirect-call occurrence, with entry-state-dependent target |
| F-dep-002 | Grouping label for remaining unresolved pre-mutation call occurrences |
| F-dep-003 | Final unresolved indirect transfer |

F-dep-002 does not identify a shared implementation. Each occurrence’s target identity, calling convention, implicit inputs, additional results, memory effects, ownership effects, failure behavior, and return behavior remain **documented_only**.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Explicit widths versus global widths

An explicit unsigned-byte load yielding 0xfe, widened into a word destination, supplies 254.

If the explicit narrowing before the 0x67 assignment produces 0xab, that is the supplied right-hand value. It does **not** establish that the global assignment touches exactly one byte or leaves adjacent storage unchanged.

A 32-bit anchor capture likewise does not require a four-byte source read.

### Captured-global base

If the 0x05 global expression and its conversion produce unsigned value 0x80, its offset-0x54 word load addresses 0xd4, subject to validity.

A raw byte of 0x80 at 0x05 alone is insufficient evidence for that capture. Results above 255 must not be truncated on an assumed byte-source basis. The capture precedes the 0x01 assignment.

### Ordering and saved captures

An access-sensitive trace retains the 0x54 global assignment between the shifted-primary-source store and the two loaded-base stores. It retains the later 0x15 assignment after both scaled-base byte stores.

All four working-offset-0x20 word stores precede the anchor capture, which precedes the 0x60 assignment.

If conversion of the 0xa9 expression produces saved representation 0x123456ab, the later explicit narrowing supplies 0xab to the 0x67 assignment even if intervening stores change the 0xa9 source.

### Repeated global observations

The three subsequent 0x67 reads remain distinct. If declarations and memory semantics permit intervening aliases to change that global, later observations may differ.

A saved working-offset-0x54 representation of 0x123456ef supplies 0xef to the third later working-offset-9 store under low-eight-bit narrowing, independently of the current 0x67 value.

### Character arithmetic and shifts

If character conversion of the adjusted base yields 254, the expression supplied to the 0x3a global is 2080. If that conversion instead yields −2, the expression is 32. The global’s declaration determines subsequent conversion; no universal byte result is specified.

Bit-vector working-base low-byte shift counts include 8 and 24, but also counts at least 32. No portable result or implicit count masking is promised for an out-of-range 32-bit signed shift.

## Uncertainties

The following remain **documented_only**:

- Application purpose, safe entry, and actual calling convention.
- Dependency identities, preservation, implicit results, and lifetime effects.
- Final-transfer return behavior and control-flow fidelity.
- All undeclared globals’ types, access widths, signedness, and type-dependent conversions.
- Memory mapping, alignment, byte order, aliasing, volatility, and concurrency.
- Operand-level evaluation order not independently sequenced.
- Signed arithmetic, character conversion, and architecture-specific shift behavior.
- Numeric repeated-block cardinality as an executable acceptance requirement.
- Validity and ownership of address-bearing values.

These limitations do not make established operations or ordering optional. Field-shaped manipulation is **inferred intent**, not proof of object layout or persistent structure.

## Evidence

The supplied static representation is the behavioral evidence. Comments, prior prose, and feedback are not independent authority.

Explicit pointer accesses establish their represented widths. Missing global declarations prevent the same conclusion for global expressions, including the anchor, 0xa9 capture, fixed-global assignments, and all three subsequent 0x67 reads. Destination-local widths and explicit right-hand narrowing do not resolve those global access widths.

The representation establishes the capture before the 0x01 assignment, the ordered 0x54 and 0x15 assignments, anchor timing between the working-store sequence and the 0x60 assignment, reuse of the saved 0xa9 capture, and three separately timed subsequent 0x67 reads.

No tools, external implementation access, runtime execution, or runtime tests have been used. This contract does not establish a complete executable replacement or safe invocation recipe.
