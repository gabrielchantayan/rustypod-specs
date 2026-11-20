<!-- Trusted provenance: F-e7f06dec4b3b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7f06dec4b3b

## Purpose

F-e7f06dec4b3b performs an extended sequence of dependency calls, byte-selected writes, and 16-bit and 32-bit memory accesses. Its application-level purpose is not established.

The supported behavioral model is an **ordered memory-and-call trace conditional on execution reaching each operation**. It is not a self-contained transformation of its explicit argument. Implicit entry state, unresolved dependency results, shared memory, and low numeric addresses influence execution.

This contract distinguishes:

- **Established static evidence:** accesses, value relationships, and ordering represented in the supplied material.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior for which no implementation requirement is inferred.

The represented control flow is incomplete and includes instruction-decoding warnings. No runtime tests have been performed.

## Interface

### Explicit argument and result

The visible interface has one nominally signed, 32-bit integer argument. Its first represented use is a left shift by two, supplied to an indirect dependency after the initial selector read and byte write.

No explicit result value is represented. This does not establish normal return: the visible endpoint is a control-flow truncation rather than an ordinary return.

No status code, error object, success flag, or explicit output-buffer parameter is established.

### Implicit inputs

Execution also depends on:

- An entry address used to read a byte at offset `0x01`.
- An entry byte written through the resulting selector.
- An entry control-transfer value used to derive the first indirect target.
- Primary and secondary machine-result values supplied by dependencies.
- Memory used as selectors, words, addresses, and destinations.
- Shared memory at numeric addresses including `0x01`, `0x28`, and `0x7C`.
- A late shared 32-bit quantity whose address and application role are not established.

These inputs do not have a validated public calling convention. Secondary machine results must not automatically be treated as ordinary language-level return values.

### Widths and arithmetic

| Quantity | Supported treatment |
|---|---|
| Selector | Unsigned 8-bit read, zero-extended when used arithmetically |
| Byte store | Low 8 bits of the stored expression |
| Halfword store | Low 16 bits |
| Word access | 32 bits |
| Paired result | Represented 64-bit quantity, separated into lower and upper 32-bit portions |
| Unsigned arithmetic | 32-bit unless an explicitly wider operation is represented |

Unsigned right shifts are logical. Unsigned 32-bit multiplication, subtraction, and left shifts retain the low 32 bits. Masks apply before their associated shifts.

Some intermediate values and address expressions are nominally signed. Negative left shifts, signed overflow, and conversions between signed and unsigned forms do not have an independently established portable contract. Bit-vector conclusions involving these intermediates are explicitly conditional below.

### Preconditions and failures

Reaching the complete represented trace requires executable call targets, readable source addresses, writable destinations, suitable implicit machine state, and dependencies that return sufficiently to continue.

These are execution requirements, not validations performed by the routine. No null, bounds, alignment, argument-range, or address-mapping checks are visible.

If an access or call cannot execute, the evidence establishes no particular exception, return code, recovery action, or rollback.

## Behavior

### Selector-mediated byte accesses

A recurring access reads an unsigned byte from a computed source address, then writes one byte to the absolute numeric address equal to that selector plus `0x11`.

The destination therefore lies in `0x11`–`0x110`, inclusive. It is not established as a field relative to the selector’s source object.

Each selector read precedes its corresponding write. Later operations can reread the same source address and obtain another selector. Calls, intervening writes, or external state changes can affect those later observations. Repeated destinations and aliasing are permitted; selector uniqueness is not established.

Stored byte values include zero and narrowed forms of:

- A dependency result shifted left by two.
- An unsigned result shifted right by 22.
- A result masked to 30 bits, then shifted right by 20.
- A result masked to 22 bits, then shifted right by 12.
- A separately sampled word.
- A portion of a paired result.

Narrowing is significant. Masking to 30 bits and shifting right by 20 leaves up to ten bits; the byte store retains only eight.

### Entry and early retained result

The first represented memory action reads the entry selector at offset `0x01`. The next writes the implicit entry byte to the selector-derived destination.

Only afterward is the explicit argument shifted left by two and passed to the first indirect dependency. Several further calls follow. One receives a shifted dependency result and the original selector in separate argument positions.

A subsequent 32-bit indirect result is retained and passed to F-23325cd5e50c. Its retained numeric value is subsequently reused; this does not imply that the dependency preserves globals or memory.

After further indirect execution, a selector is read at offset `0x01` from a returned address, and its selected destination is cleared.

The retained result also participates in:

- A selector source formed by multiplying it by 16 and adding one.
- A byte value formed by masking it to 30 bits and shifting right by 20.
- Another selector source formed by an unsigned right shift by nine, followed by addition of one.
- A call receiving the selector-minus-retained-result difference twice.

The subtraction has unsigned operands and can wrap modulo \(2^{32}\).

### Early zero-derived operations and publication

Several retained intermediates are explicitly initialized to zero. Their represented shifts and multiplications remain zero. Intervening calls are not represented as assigning replacement values to those retained intermediates.

Consequently, some selector sources reduce to address `0x01`. Each represented read remains a fresh observation rather than a cached selector.

Other selectors depend on secondary machine results, including four-times-scaled values and masked, shifted values. Their validity as addresses is not checked.

A later indirect result is written as a 32-bit word to address `0x7C`. That write precedes a call receiving the retained result in two argument positions.

Further calls use an unsigned right shift by 11 and a right shift by nine followed by a left shift by 25. Under 32-bit unsigned arithmetic, the latter retains original bits 9–15 in result bits 25–31; it is not an unbounded-width calculation.

### Saved words and distinct shared observations

A 32-bit word is read at offset `0x14` from a secondary-result-derived address. Its sampled value is retained across subsequent calls and later stored, shifted left by 11, at another derived destination.

A separate region accesses shared address `0x28` in three represented expressions:

1. A word read is saved for a later store.
2. A following call obtains one argument from another word-read expression at `0x28`.
3. The same call obtains another argument from a separate word-read expression at `0x28`.

The saved read precedes the call’s argument expressions. **No evaluation order between those two argument expressions is established.** They are not specified as copies of the saved sample, and all three observations need not agree.

After that call and another intervening call, the saved sample—not either argument observation—is stored at a secondary-result-derived address multiplied by four, plus `0x14`.

A subsequent dependency receives that secondary result three times. Later calls and stores continue independently of whether the shared location has changed.

### Interleaved writes and paired-result reads

The middle trace repeatedly alternates selector accesses with dependency calls. Some operations write a shifted-result byte and later clear a selector-derived destination using the same retained arithmetic base.

These are separate accesses. Because selectors are reread, the later clear need not target the earlier destination. No set-and-clear protocol is established.

One paired result supports the following ordered reads:

- A word at offset `0x08` from its upper 32-bit portion.
- A word at offset `0x0C` from that same upper portion.

The first sampled word later contributes its low byte to a selector-mediated write. The second supplies an address used for that write’s selector read, after intervening calls.

The pair’s lower portion is retained independently. A later word read at offset `0x04` from that retained lower value contributes another stored byte.

These accesses establish offset relationships, not complete structure layouts, allocation sizes, or ownership. Intervening dependencies may change the original memory without replacing already sampled numeric values.

### Mapped dependency order

The mapped dependencies occur in this relative order:

1. F-23325cd5e50c.
2. F-bbb54b5f036d.
3. F-f80b3ac065fc.
4. F-de15789688a8.
5. F-ea83f9a17404.
6. F-674e75a953e8.

They are not adjacent. Other direct and indirect calls intervene.

F-bbb54b5f036d, F-de15789688a8, F-ea83f9a17404, and F-674e75a953e8 each receive one retained or derived value in two argument positions.

F-f80b3ac065fc receives four represented arguments: a paired result’s lower portion, its upper portion, and two further copies of the lower portion.

### Narrow stores and low addresses

In a later region, a retained index is explicitly zero. The represented trace then includes:

- A 32-bit store at the index multiplied by eight: address `0x00`.
- After intervening calls and byte accesses, a 16-bit store at an address derived by masking and shifting that index: also `0x00`.
- A later byte store whose destination includes four times the zero index and a separately derived base.

The word store precedes the halfword store. The halfword writes only 16 bits. Interpreting the resulting overlapping bytes requires target byte order, which is not established.

These stores are not omitted merely because address zero may be invalid in a conventional environment. Successful execution there remains **documented_only**.

Another paired result causes its lower portion’s low byte to be written at offset `0x1C` from its upper portion.

A further call has two separate argument expressions reading byte address `0x01`. As with other repeated shared-memory expressions, equality and an argument-expression evaluation order are not established.

### Consecutive stores and distinct scaling relationships

A later destination formed from a right-shifted retained value plus `0x34` receives two consecutive 32-bit stores:

- A zero-extended selector.
- That selector shifted left by one.

The second value is at most 510. Both stores are represented events.

Another destination at offset `0x34` receives the same shifted dependency result twice consecutively. The duplicate writes likewise remain distinct.

Later word destinations repeatedly use both four-times-derived values plus `0x14` and sixteen-times-derived values plus `0x14`. They are not interchangeable layouts. Some subsequent reads use the four-times-derived address even when the preceding store used the sixteen-times-derived address.

Stored values include retained results, secondary results, right-shifted words, zero, and scaled values also used in address formation.

### Late shared state: two reads with separate uses

The late shared 32-bit quantity is observed **twice in successive represented read expressions**:

- The first observation is saved for later masked arithmetic.
- The second observation is multiplied by four to form a retained address-related value.

No single-snapshot guarantee is established. The second calculation does not use the saved first observation.

Following intervening calls and a word store, the second observation’s four-times-scaled value supplies a selector source at that scaled value plus one. A returned result shifted left by two supplies the stored byte.

The first saved observation separately supplies a call argument obtained by masking to 30 bits and shifting right by ten.

After further calls, the selector source based on the second observation’s scaled value is read again. Its selected destination receives a byte obtained from a retained dependency result masked to 30 bits and shifted right by 20.

### Late shift, word reads, and zero-derived selectors

The second observation’s four-times-scaled value is subsequently shifted left by 26.

Under **32-bit bit-vector arithmetic**, this retains the second observation’s lowest four bits in result bits 28–31. It is not necessarily zero. Equivalently, its value is the second observation’s low nibble multiplied by `0x10000000`.

A later call has two separate word-read argument expressions at this shifted value plus `0x1C`. The evidence does not establish an evaluation order between those expressions or require equal observations.

Multiplying the shifted value by `0x400`, however, discards every remaining set bit under the same 32-bit model and produces zero. Subsequent selectors based on that product plus one therefore read address `0x01`.

After intervening calls, one such selector read clears its selected destination; a later fresh read supplies a destination for a narrowed secondary-result byte. These remain separate observations.

The arithmetic simplifications are conditional on the bit-vector model because nominally signed intermediates occur in this region. They do not resolve portable signed-language behavior.

### Continuation and endpoint

F-674e75a953e8 occurs after the late shared-state region, not at the endpoint.

Subsequent operations include word stores using four-times-derived destinations plus `0x14`, selector writes using scaled retained values, more calls, and a zero-valued word store at a paired upper-result-derived destination plus `0x14`.

The later trace also includes a destination formed by multiplying a retained unsigned value by `0x10000000` and adding `0x14`; its stored value is that retained value multiplied by `0x4000000`.

Repeated later groups store a four-times-scaled result at a sixteen-times-derived destination plus `0x14`, then use further calls and four-times-derived destinations. Some groups read words from the four-times-derived address plus `0x14`. Where two such reads appear as separate call arguments, they are not established as one snapshot.

The final group writes a shifted-result byte through a selector, performs additional calls, rereads the selector source, writes a narrowed masked high-bit value, and performs further calls. The representation then truncates. Normal return, continuation, and actual endpoint hardware behavior are not established.

## Side effects

Directly represented modifications include:

- Byte writes within `0x11`–`0x110`.
- A word write at `0x7C`.
- Separate word and halfword writes at zero.
- Dependency-derived byte writes, including an offset-`0x1C` write.
- Word writes at scaled destinations with offsets `0x14` and `0x34`.

Dependencies may have additional effects. No global, memory, device-state, or shared-state preservation guarantee is established.

No allocation, release, reference-count change, ownership transfer, or buffer-lifetime obligation is independently identifiable. Repeated arguments do not establish copying, retention, or release.

No rollback, synchronization, atomicity, reentrancy, or thread-safety guarantee is represented. Ordered stores in this contract describe the static trace, not an independently established hardware memory-ordering rule.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-23325cd5e50c | Receives a retained 32-bit indirect result near the beginning |
| F-bbb54b5f036d | Receives one dependency-derived value twice |
| F-f80b3ac065fc | Receives lower and upper paired-result portions, followed by two lower-portion copies |
| F-de15789688a8 | Receives one retained result twice |
| F-ea83f9a17404 | Receives one result twice |
| F-674e75a953e8 | Receives one result twice before further stores and calls |

Their purposes, complete ABIs, ownership effects, failure behavior, preservation rules, and normal-return guarantees are **documented_only**.

Other represented dependencies lack supplied opaque IDs. Their identities and equivalence relationships remain **documented_only**; this contract does not invent mappings or omit their possible effects.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Selector boundaries.** A selector of zero chooses `0x11`; a selector of 255 chooses `0x110`. Neither is sign-extended.

2. **Byte narrowing.** A 32-bit unsigned value shifted right by 22 contributes only original bits 22–29 to the stored byte. Original bits 30–31 are discarded by narrowing.

3. **Unsigned wrapping.** At an unsigned 32-bit scaling operation, shifting `0x40000001` left by two produces `0x00000004`. This does not define every nominally signed shift.

4. **Fresh selector.** If a dependency changes byte `0x01`, a later represented read uses that later observation, not an earlier cached selector.

5. **Saved shared word versus arguments.** Suppose the saved read at `0x28` observes `0x11111111`. If the two subsequent argument reads observe `0x22222222` and `0x33333333`, those differing observations do not violate the contract. Their evaluation order is unspecified. The later saved-sample store still writes `0x11111111`.

6. **Separate late observations.** If the first late shared-state read observes `0x00000400` and the second observes one, the first supplies a masked-right-shift call argument of one. The second supplies a scaled selector base of four, hence selector source address five.

7. **Nonzero late word address.** If the second late observation is one, its four-times-scaled value is four. The later 26-bit left shift produces `0x10000000` under the bit-vector model. Both word-read expressions address `0x1000001C`, not `0x1C`.

8. **Late nibble boundary.** Under that model, a second observation of 15 produces shifted value `0xF0000000`; an observation of 16 produces zero. Multiplication of either shifted value by `0x400` produces zero, supporting the later selector source `0x01`.

9. **Consecutive replacement stores.** A selector of seven in the offset-`0x34` pair produces a word write of seven followed by fourteen. Reporting only fourteen omits an event.

10. **Duplicate stores.** Both consecutive identical offset-`0x34` writes remain trace events.

11. **Low-address effects and failure.** If the zero-derived word and halfword stores are reached, both target zero. If the environment rejects an access, no particular error or rollback is required by this evidence.

## Uncertainties

The following remain **documented_only**:

- Completeness and accuracy of represented control flow.
- Actual entry convention and implicit machine state.
- Secondary-result ABI and dependency signatures.
- Dependency effects, memory ownership, and preservation.
- Signed arithmetic outside representable ranges.
- Target byte order, alignment, and address mapping.
- Whether low addresses represent ordinary memory, devices, or invalid locations.
- Timing and consistency of repeated shared-memory observations.
- Object sizes, layouts beyond observed offsets, and lifetimes.
- Normal return from dependencies and behavior beyond truncation.

Construction, encoding, or dispatch interpretations are possible **inferred intent**, not acceptance requirements.

## Evidence

This contract uses the supplied static representation and opaque dependency map. Established support includes access widths, selector zero-extension, masks, retained samples, separate shared-read expressions, relative call order, repeated stores, and the absence of an ordinary represented return.

The late arithmetic distinction follows 32-bit width analysis: four-times scaling followed by a 26-bit left shift retains four original low bits; the subsequent multiplication by `0x400` removes them. The first and second late shared reads feed different uses. Likewise, the saved `0x28` sample is distinct from the following call’s two shared-read expressions.

Confidence is limited by reported decoding problems, removed unreachable control flow, overlapping shared symbols, unresolved calls, implicit inputs, and terminal truncation.

No runtime execution, dependency implementation inspection, or independent instruction validation has been performed.
