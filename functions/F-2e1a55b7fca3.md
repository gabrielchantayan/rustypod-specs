<!-- Trusted provenance: F-2e1a55b7fca3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e1a55b7fca3

## Purpose

F-2e1a55b7fca3 performs a long sequence of dependency calls and direct memory accesses. Its application-level purpose is not established. The available evidence supports an access-and-state contract, not a characterization as initialization, decoding, serialization, allocation, or release.

The operation consumes execution state beyond its two declared inputs, accesses numeric address zero and other low addresses, and has no demonstrated normal return. This contract distinguishes:

- **Established static evidence:** represented accesses, widths, data relationships, and sequencing.
- **Inferred intent:** interpretations that impose no additional requirements.
- **documented_only:** unresolved behavior or invocation requirements.

Every description of a later operation is conditional on preceding accesses completing and dependencies returning. The represented sequence is not evidence that an ordinary invocation can reach its end.

## Interface

### Declared inputs

Two inputs are declared, each 32 bits wide.

| Input | Established use |
|---|---|
| First input | No explicit use is represented. This does not establish that all corresponding incoming machine state is irrelevant. |
| Second input | Supplied as the sole explicit argument to the first dependency call, after an initial byte read and byte write. |

Neither input has an established pointer, length, handle, enumeration, or ownership meaning.

### Additional execution inputs

The initial operations also require:

- An incoming 32-bit address-bearing value.
- An incoming eight-bit value to write.
- Another incoming 32-bit value from which bits are extracted.
- Incoming execution-context state that contributes to the first dependency target.

Later operations consume direct dependency results and secondary machine outputs. Several results are represented as 64-bit aggregates, with their low and high 32-bit portions used separately. Whether these are genuine 64-bit return values or combined machine-result channels is **documented_only**.

A complete calling convention cannot be reconstructed from the declared inputs alone.

### Outputs and completion

No ordinary return value is declared. Observable outputs include direct memory mutations, dependency arguments, and dependency effects.

The representation terminates at an invalid-instruction boundary without a demonstrated normal epilogue. Normal return, deliberate trapping, nonreturning execution, and behavior beyond that boundary are **documented_only**.

### Preconditions and ownership

Every reached address must support the represented access width and any target-specific alignment requirement. This includes numeric addresses zero, `0x1`, `0x28`, and `0x7c`, as well as computed addresses. No explicit validity checks precede these accesses.

No allocation, release, ownership transfer, or borrowing convention is established. An address-like argument does not prove that its recipient owns or releases memory. Unknown dependencies must not be assumed to preserve memory, globals, or unspecified machine state.

## Behavior

### Widths, arithmetic, and narrowing

The operation represents eight-bit reads and writes, 16-bit writes, and 32-bit reads and writes. Byte selectors are unsigned values in the range 0–255.

Byte writes retain the low eight bits; halfword writes retain the low 16 bits. A signed-looking conversion before a byte write does not establish a signed application field.

Explicit unsigned 32-bit arithmetic is modular: multiplication, subtraction, and left shifts retain only the low 32 bits. Unsigned right shifts introduce zeros. Expressions involving signed intermediates do not independently establish portable overflow behavior; their exact machine interpretation is **documented_only** where relevant. Endianness is not established.

The following byte extractions are significant:

| Transformation before narrowing | Bits represented in the written byte |
|---|---|
| Unsigned word shifted right by 22 | Bits 22–29 |
| Word masked with `0x7fffffff`, then shifted right by 21 | Bits 21–28 |
| Word masked with `0x3fffffff`, then shifted right by 20 | Bits 20–27 |
| Word masked with `0x003fffff`, then shifted right by 12 | Bits 12–19 |
| Word shifted left by two | Original bits 0–5 in positions 2–7; low two bits zero |
| Unsigned 64-bit aggregate shifted right by 54 | Bits 54–61 |

These are not interchangeable with extracting a conventional most-significant byte.

### Byte-selected destinations

Many writes first read an unsigned byte from a computed source address, then use that byte’s numeric value plus `0x11` as the address of an eight-bit write.

Such destinations range from `0x11` through `0x110`, inclusive. No separate destination-buffer base is represented.

Selector-source addresses include an incoming value plus one, scaled dependency results plus one, and masked or shifted values plus one. Visible scales include 2, 4, 8, 16, 64, 1024, 4096, and larger powers of two.

Each represented selector read remains a separate observation. Intervening calls or aliased writes may change later observations. This rule does not imply an order between separate argument expressions within one call.

### Entry and early propagation

The initial byte selector is read one byte beyond the incoming address-bearing value. The incoming eight-bit value is written to the selected destination before the first dependency call, which receives the second declared input.

Additional calls follow, including one receiving the incoming address-bearing value logically shifted right by 11. After further calls, the initial selector-source address is read again. That new selector determines the destination for a byte extracted from the other incoming word using the `0x7fffffff` mask and right shift by 21.

Subsequent groups propagate call results scaled by four. Some calls combine a scaled result with a retained selector. Both argument arrangements occur: scaled result followed by selector, and selector followed by scaled result.

An early result is retained across calls and supplies later scaled arguments and a byte extracted after a right shift by 22. A separately established zero causes a selector-source calculation to resolve to numeric address `0x1`; the selected destination receives zero. This is a represented low-address access, not evidence of an omitted buffer base.

### First mapped dependency and low-address state

F-23325cd5e50c receives one 32-bit result. Its visible direct return is unused, but the argument is retained for later scaling, masking, subtraction, and extraction. A subsequent call receives two copies of the difference between a retained selector and that retained result.

Further call groups alternate with selector reads and byte writes, including zero writes.

A later dependency result is stored as a 32-bit word at numeric address `0x7c`, before a call receives that same captured result in both argument positions.

Another region reads a 32-bit word at offset `0x14` from a secondary-result base. That captured word is supplied twice to a call and retained for later arithmetic and stores.

### Three separate word observations at numeric address `0x28`

After a preceding call group, the operation reads a 32-bit word at numeric address `0x28` and captures it for a later store.

The next call has two argument expressions, each containing a separate 32-bit read of numeric address `0x28`. Thus, this region represents **three word reads**, not one read reused three times:

- The captured read occurs before both argument reads.
- Each argument is supplied from its own observation.
- The evidence does not establish which argument expression is evaluated first.
- Neither argument observation is required to equal the captured observation or the other argument observation.

After that call, another call with no explicit arguments occurs. The captured first observation—not either argument observation—is then stored at an address formed as four times a secondary result plus `0x14`.

This distinction remains relevant even though no explicit write to `0x28` appears between the three reads: read-sensitive memory and concurrent modification are unresolved, not excluded.

### Aggregate results and captured fields

One represented 64-bit result supplies a high 32-bit base and a retained low 32-bit value. A word at base plus `0x8` is read first; a word at the same base plus `0xc` is read next. Both reads precede the call receiving two copies of the retained low portion.

After another call, the low byte of the captured offset-`0x8` word is written to a destination selected by reading a byte one beyond the address supplied by the captured offset-`0xc` word.

Elsewhere, a word at offset `0x4` from a retained base is captured before calls; its low byte is written afterward. This is not a fresh post-call field read.

Additional aggregate-derived accesses include an eight-bit write at the high-portion base plus `0x1c`, using the low byte of the aggregate. Another region retains the aggregate’s low portion plus `0xc`, later reads a word four bytes beyond that derived address, and uses the captured word as a selector-source base.

These isolated offsets do not establish a complete record layout.

### Middle and later mapped calls

The mapped dependencies occur in this order, with numerous other calls and accesses between them:

1. F-23325cd5e50c.
2. F-bbb54b5f036d.
3. F-f80b3ac065fc.
4. F-de15789688a8.
5. F-ea83f9a17404.
6. F-674e75a953e8.

F-bbb54b5f036d receives two copies of one captured word. F-f80b3ac065fc receives four arguments: low aggregate portion, high aggregate portion, low portion, low portion. The remaining three mapped calls each receive two copies of a captured word.

Two other direct dependencies have no supplied opaque IDs. One receives a low/high/low/low aggregate arrangement; the other receives two copies of a captured word. Their identities remain **documented_only**.

### Separate byte reads used as call arguments

After a selector-based write using a zero-derived masked-and-shifted base, a propagation call receives a scaled result and the retained selector. A call with no explicit arguments follows.

The next call reads the byte at numeric address `0x1` separately in each of its two argument expressions. These are two eight-bit observations used directly as arguments, not one retained selector supplied twice. Their argument-position association is preserved, but their evaluation order and equality are not established.

After this two-argument call, another call with no explicit arguments occurs. Only then does the operation read a selector from an address formed as a retained aggregate low portion multiplied by `0x02000000`, plus one, and write zero to the selected destination. Subsequent calls consume a secondary result, followed by another selector read from that scaled source.

The argument reads must therefore remain distinct from both earlier selector reads and the later selector reads.

### Zero-derived stores and overlapping low memory

A later region establishes a 32-bit zero and stores a dependency result as a word at eight times that zero: numeric address zero.

After intervening calls and selector-based writes, a destination formed by masking that retained zero with `0x7fffffff` and shifting right by 11 receives a 16-bit store. Its address is also zero. The value is the low halfword of a dependency result.

The word and halfword stores are separate effects with different widths. The word store at zero overlaps the byte at numeric address `0x1`; the later halfword store also covers that byte. Subsequent observations of `0x1` cannot be treated as independent of these writes.

Another eight-bit store uses a result-dependent base, formed by masking a retained aggregate low portion with `0x3fffffff` and shifting right by seven, plus four times the retained zero. Only the added offset vanishes.

### Repeated stores and scaled word locations

Two pairs of repeated 32-bit stores occur at offset `0x34`:

- A destination formed from a retained value shifted right by nine receives a widened selector, then twice that selector.
- A secondary-result base receives the same shifted result twice.

No dependency call intervenes within either pair. Both represented writes remain part of the access trace.

Numerous other word stores use four times a result plus `0x14`; some instead use sixteen times a result plus `0x14`. A later store uses a result multiplied by `0x10000000`, plus `0x14`, and writes that result multiplied by `0x04000000`. These layouts and scale factors are not interchangeable.

Some stores precede calls receiving the corresponding four-times-scaled value. Other stores propagate captured fields, secondary results, zero, or masked-and-shifted words. No object, link, or table interpretation is established.

### Two observations of the late global word

A late word store at four times a secondary result plus `0x14` is followed by **two separate 32-bit reads** of the same unnamed global location:

- The first observation is retained. After intervening calls and writes, it supplies arithmetic that masks with `0x3fffffff` and shifts right by ten.
- The second observation is multiplied by four. That derived value supplies subsequent selector-source addresses and further address calculations.

The first read precedes the second. No dependency call or direct write is represented between them, but equality is not established. They must not be collapsed into a shared snapshot.

The second-derived scaled value is used for a selector read before a byte write, and is read through again after further calls for another selector-based write. The first observation remains the source for the separate masked-and-shifted argument.

### Separate word argument reads at derived offset `0x1c`

Later in this same global-derived region, the value obtained by multiplying the **second** global observation by four is shifted left by 26. The resulting derived base identifies a word-read address at offset `0x1c`.

After a propagation call receiving a scaled result and retained selector, and then a call with no explicit arguments, the next call contains two separate 32-bit reads at that derived base plus `0x1c`, one in each argument expression. No shared captured read is represented. Evaluation order between the expressions is not established.

After this call, another no-explicit-argument call occurs. A byte is then read from one beyond the derived base multiplied by 1024, and zero is written to the selected destination.

Under 32-bit modular arithmetic, multiplying the shifted base by 1024 yields zero. This does not make the preceding offset-`0x1c` reads disappear, nor does it make their address necessarily `0x1c`. Exact treatment of signed-looking intermediate shifts remains **documented_only**.

### Remaining propagation and ending

After F-674e75a953e8, the operation continues through scaled word stores and selector-based byte writes.

One later call contains two independent word reads at a four-times-scaled base plus `0x14`: one read supplies an argument shifted right by 12, while the other supplies an argument shifted left by two. Their evaluation order and equality are not established. Elsewhere, a word from such an address is explicitly captured before calls and later stored after a right shift by 12.

The final visible region again alternates calls with selector-based byte writes. It ends at the invalid-instruction boundary without an established final-state or successful-return guarantee.

## Side effects

Direct effects include low-address memory mutations, dependency-derived writes, and byte-selected writes. The footprint is not bounded by the two declared inputs.

Aliasing matters across widths and regions. A word write can affect a later byte selector; a dependency can modify memory or globals before later observations. Captured values and fresh reads have different contracts.

Repeated reads and writes are represented access events. Whether individual events have device-visible significance is **documented_only**; no ordinary-memory-only restriction is established.

There is no demonstrated transaction, rollback, synchronization, cleanup, or reentrancy guarantee. No explicit validation, error code, or recovery branch is represented. Invalid accesses or dependency failures have no established recovery behavior, and partial effects may already have occurred.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-23325cd5e50c | One 32-bit argument; visible direct return unused. |
| F-bbb54b5f036d | Two copies of one captured 32-bit value; visible direct return unused. |
| F-f80b3ac065fc | Four arguments in low/high/low/low aggregate order. |
| F-de15789688a8 | Two copies of one captured 32-bit value; visible direct return unused. |
| F-ea83f9a17404 | Two copies of one captured word; visible direct return unused. |
| F-674e75a953e8 | Two copies of one captured word; visible direct return unused. |

Other calls use execution-context-derived or fixed-position-derived targets, alongside the two unidentified direct dependencies. Complete signatures, state preservation, memory effects, ownership, and exceptional behavior are **documented_only**.

An unused visible direct return does not establish absence of effects or secondary outputs.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Entry ordering.** A first selector of `0x20` causes the incoming byte to be written at numeric address `0x31` before the first dependency call.

2. **Fresh selector.** If intervening effects change that selector to `0x40`, the later explicit reread selects `0x51`, not `0x31`.

3. **Three observations at `0x28`.** Suppose the captured read obtains `0x11111111`, while the argument reads obtain `0x22222222` and `0x33333333` in their respective positions. The call receives the latter two values; the later store uses `0x11111111`. No left-to-right argument evaluation requirement follows.

4. **Repeated byte arguments.** If the two argument observations at `0x1` produce different bytes, each argument must reflect its own observation. A single read duplicated into both positions does not preserve the represented accesses.

5. **Distinct global observations.** If the first global read is `0x400` and the second is 3, the later masked-and-shifted argument is 1, while the second-derived scaled value is 12 and its selector-source address is `0xd`.

6. **Derived word arguments.** If the two reads at the derived base plus `0x1c` differ, each supplies its own argument. Neither may be replaced with the earlier global observation or a single cached field read.

7. **Narrowing.** Shifting unsigned `0xffc00000` right by 22 and narrowing to a byte produces `0xff`. A halfword store supplied with `0x12345678` writes the 16-bit pattern `0x5678`.

8. **Repeated writes.** A selector of 7 in the unequal offset-`0x34` pair produces a word write of 7 followed by a word write of 14.

9. **Captured field.** A captured offset-`0x4` word of `0x11223344` supplies byte `0x44` later, rather than the low byte of a fresh post-call read.

10. **Completion.** Reaching the final visible call does not establish normal return.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid invocation context.
- Complete calling convention and secondary-result interpretation.
- Contracts and identities of unmapped dependencies.
- Endianness, alignment, read-sensitive memory, and concurrency semantics.
- Exact signed-intermediate arithmetic semantics.
- Completeness of represented control flow and behavior beyond its ending.
- Allocation, ownership, release, synchronization, and reentrancy rules.

**Inferred intent:** repeated scaled stores may represent structured-state propagation. This does not establish a record model or justify combining accesses.

## Evidence

This contract uses the supplied static representation as evidence, not the prior draft or embedded comments as independent authority.

The representation supports three distinct word reads at `0x28`, two ordered observations of the late global word, separate byte argument reads at `0x1`, and separate word argument reads at the derived offset `0x1c`. It does not establish an evaluation order between the paired argument expressions.

No runtime execution, smoke run, or tests have been performed. Acceptance examples describe conditional consequences of represented operations, not observed results.
