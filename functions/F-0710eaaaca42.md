<!-- Trusted provenance: F-0710eaaaca42; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0710eaaaca42

## Purpose

F-0710eaaaca42 performs an ordered population of unresolved indirect invocations, retains quantities derived from their results, and propagates those quantities through mixed-width memory accesses. Its final represented action before return is another unresolved indirect invocation.

The established behavior concerns control transfers, value dependencies, memory accesses, and their represented order. The supplied material does not establish an application-level purpose, text format, object model, or allocation layout. Repeated offsets and character-like constants do not establish those interpretations.

This contract distinguishes:

- **Established static evidence:** explicit operations, operand widths, conversions, ordering, and saved-value dependencies in the supplied representation.
- **Inferred intent:** possible interpretations that impose no acceptance requirement.
- **documented_only:** behavior or contract detail not resolved here.

No runtime tests have been performed.

## Interface

### Explicit inputs and outputs

| Input | Represented width | Visible contribution |
|---|---:|---|
| First argument | 32 bits | No explicit use. |
| Second argument | 32 bits | No explicit use. |
| Third argument | Signed 32 bits | An early indirect invocation receives this value shifted left by six. |

Unused displayed arguments are not established as irrelevant to unresolved dependencies: the complete calling convention and incoming machine-state dependencies are unknown.

There is no represented explicit return value, status code, or output-buffer parameter. Observable outputs include dependency invocations and memory-access events. A return is represented after the terminal invocation; its returned-register contents are unspecified.

### Implicit inputs and preconditions

Execution additionally depends on incoming control-link state, indirect destinations, primary and secondary result channels, represented 64-bit results, fixed globals, and storage reached through calculated or loaded addresses.

Normal completion requires preceding dependencies to return, required accesses to succeed, and arithmetic and address conversions to have supported target behavior. Required storage may include low absolute addresses. Alignment requirements, accessible extents, and valid allocation boundaries are not established.

No comprehensive validation or explicit rejection of zero addresses is visible. In particular, the accessible extents required for undeclared fixed globals cannot be inferred from the widths of surrounding temporaries.

## Behavior

### Contract boundary and invocation phase

The invocation phase has no visible input-dependent branch or loop. Its first destination derives from incoming control-link state plus `0xFFE`; subsequent destinations derive from represented control-location anchors plus the same displacement.

Displayed argument counts range from zero to four. Arguments include zeros, shifted primary results, secondary result channels, saved earlier results, and separated halves of 64-bit results. A displayed zero-argument invocation does not establish that other argument channels are cleared or ignored.

Common transformations include left shifts by eight, other left shifts through 31 positions, masks retaining low-order fields, and unsigned right shifts. Examples include:

- Low 20 bits shifted right by ten: a 10-bit field.
- Low 17 bits shifted right by four: a 13-bit field.
- Low 24 bits shifted right by 19: a 5-bit field.
- Low 18 bits of a high result half shifted right by eleven: a 7-bit field.
- Unsigned 64-bit right shifts by 34, 36, or 44.

An unused primary result does not make an invocation removable. Dependency side effects, secondary outputs, faults, and termination remain relevant or unresolved.

**Scope limitation:** this contract does not enumerate every intervening invocation, its argument tuple, or every intermediate result transformation. That exhaustive invocation schedule remains **documented_only**. The direct-mutation description below gives ordered phases; the expanded publication region also has an explicit count limitation.

### Widths and fixed-global access rules

Explicit computed-address byte, halfword, and word accesses have widths of 8, 16, and 32 bits. Represented wide results have width 64 bits. Explicit unsigned-byte loads are zero-extended when promoted to unsigned words. Explicit byte and halfword stores retain the low eight and sixteen bits.

**Named fixed globals have no supplied declarations.** Their names, assignments to 32-bit temporaries, explicitly narrowed assignment values, and overlapping-symbol comments do not independently establish storage access widths.

| Absolute location | Established value-level operation | Unresolved storage properties |
|---|---|---|
| `0x01` | Assigned the small base explicitly narrowed to eight bits | Store width and destination conversion |
| `0x05` | Read and converted to unsigned 32 bits | Load width, signedness, extension and interpretation |
| `0x15` | Assigned the small base explicitly narrowed to eight bits | Store width and destination conversion |
| `0x21` | Assigned a fresh value read from `0x67` | Both globals’ types and assignment conversion |
| `0x3A` | Assigned a narrowed-source arithmetic expression | Store width and destination conversion |
| `0x54` | Assigned the saved old word shifted left by one | Store width and destination conversion |
| `0x60` | Assigned `0x20` | Store width and destination conversion |
| `0x67` | Assigned an eight-bit-narrowed saved value; subsequently reread | Store/load widths, signedness, extension and conversion |
| `0xA9` | Read into a 32-bit saved quantity | Load width, signedness and extension |
| `0x087FBBBC` | Read into a signed 32-bit saved anchor | Load width, signedness and conversion |

Consequently, these are global read or assignment events, not unconditionally byte or word accesses. This limitation does not apply to separately explicit pointer accesses whose computed addresses happen to equal one of these locations.

Signed left shifts, signed scaling, and address formation may overflow. A universal wrapping rule is not supplied. Unsigned right shifts insert zeros. Low-bit arithmetic examples below are conditional, not target guarantees.

### Saved quantities

The following semantic labels identify value dependencies, not object types:

| Quantity | Derivation |
|---|---|
| Base seed | An earlier signed returned word |
| Primary base | Base seed multiplied by \(2^{25}\) |
| Shifted address | A saved low result half shifted left by 25 |
| Small field | A returned word shifted left by eight, masked to low 15 bits, then logically shifted right by six |
| Small base | Twice the small field |
| Saved shifted word | A separate earlier returned word shifted left by eight |
| Final halves | Low and high 32-bit portions of the final wide result preceding direct mutation |

Under 32-bit bit-vector semantics, the small field ranges from zero to `0x1FC` in multiples of four; the small base ranges from zero to `0x3F8` in multiples of eight.

### Initial publication and fixed-global capture

The mutation prefix publishes the shifted address as a word at final-high-half plus `0x14`. It then writes the high half’s low byte at shifted-address plus `0x09`, publishes the primary base as a word at final-low-half multiplied by 256 plus `0x54`, and overwrites the same shifted-address-relative byte with the shifted address’s low byte.

The primary-base word at `0x14` is cleared before an explicit unsigned-byte read at primary-base plus `0x05`. The small-base word at `0x14` is then cleared, and the saved byte is zero-extended into a word at primary-base plus `0x34`.

A word read at primary-base plus `0x14` supplies the **saved loaded word**. Another small-base-`0x14` clear follows. Its saved low byte is then written twice at small-base plus `0x15`.

Next, the global at absolute `0x05` is read and converted to unsigned 32 bits, producing the **fixed-global base**. This is distinct from the preceding explicit unsigned-byte load. The converted base is not established as restricted to zero through 255.

The global at `0x01` receives an explicitly eight-bit-narrowed small base. Computed byte publications then place the small field at fixed-global-base plus `0x15` and the saved shifted word at primary-base plus `0x11`. The saved shifted word is published as a word at saved-loaded-word plus `0x54`; the primary-base-`0x11` byte is then cleared.

A word read at fixed-global-base plus `0x54` supplies the **saved old word**, after which that location is replaced with the saved loaded word. Later uses retain this snapshot and the captured fixed-global base rather than rereading their original sources.

### Ordered propagation after replacement

Before reading primary-base plus `0x74`, the function publishes the small field as a **word at small-base plus `0x34`**. This intervening store can affect that subsequent read through aliasing.

The primary-base-`0x74` word supplies the **primary-linked base**. Its `0x54` word first receives the base seed shifted left by 26. Only afterward does an expression read small-base plus `0x74` separately for a destination dependency and a stored value, publishing the value at the destination plus `0x54`. The two reads are represented separately; their precise relative machine ordering within that expression is unresolved.

The following propagation retains an important overwrite:

- The saved old word is published at saved-old-word plus `0x24`.
- Its low byte is published at primary-linked-base plus `0x09`.
- Primary-linked-base plus `0x54` is overwritten with the saved shifted word.
- The same self-publication and primary-linked byte publication repeat.

Thus the primary-linked `0x54` location receives two distinct payload derivations at different times.

The small-base-`0x14` word next receives the base seed shifted left by 26. The global at absolute `0x54` is assigned the saved old word shifted left by one. The small-base-`0x14` word subsequently receives the saved old word twice. Two byte writes of four times the saved old word follow at small-base plus `0x15`, before the global at absolute `0x15` receives the explicitly narrowed small base.

These are ordered events, not merely final-state requirements.

### Mixed-width propagation

Four times the saved old word forms the **recurring base**; eight times it forms the **eightfold value**.

The recurring-base-`0x54` word first receives the saved shifted word. The small-base-`0x34` word receives the saved old word shifted left by two. A fresh small-base-`0x74` word, doubled, supplies the **doubled loaded base**. The recurring-base-`0x54` word then receives the recurring base itself.

After a byte publication of the eightfold value at recurring-base plus `0x15`, the doubled loaded base is published at saved-old-word plus `0x44`. An explicit unsigned-byte read at recurring-base plus `0x05` selects a word read at that byte value plus `0x74`, yielding the **selected base**. Its `0x44` word receives the doubled loaded base.

Further propagation overwrites recurring-base plus `0x15` with the recurring base’s low byte, publishes the selected base’s low byte at recurring-base plus `0x0D`, self-publishes the recurring base at `0x14`, and republishes the shifted saved old word at `0x54`.

A word read at recurring-base plus `0x64` yields another base. That base receives the selected base’s low byte at `0x0D` and itself at `0x14`. A separate word read at recurring-base plus `0x04` yields the **byte-source base**.

The doubled loaded base receives the selected base’s low byte at `0x09` and the eightfold value at `0x44`. An unsigned byte is then read at byte-source-base plus `0x05`. Before another read of that same byte location:

- Recurring-base plus `0x09` receives the doubled loaded base’s low byte.
- Selected-base plus `0x44` receives the zero-extended first byte.
- Recurring-base plus `0x14` receives the eightfold value.
- Doubled-loaded-base plus `0x15` receives the byte-source base’s low byte.
- Doubled-loaded-base plus `0x44` receives the recurring base.

The second unsigned-byte read selects a fresh word read at byte-value plus `0x74`, yielding the signed **constant-source word**. Before that word read, recurring-base plus `0x15` receives the eightfold value’s low byte.

Twice the constant-source word is the **tail base**; four times it is the **publication payload**. Before constant publication, recurring-base plus `0x15` receives the byte-source base’s low byte, its `0x54` word receives the shifted saved old word, its `0x09` byte receives the tail base, and its `0x14` word receives the eightfold value.

### Constants and expanded publication

Constant `0x3A` is published as an explicit byte at the publication payload, as a halfword at eight times the constant-source word, and as a word at publication-payload plus `0x20`.

The recurring base reduced by `0x96` is the **adjusted value**. Four times that value is the **adjusted base**. Its `0x20` word receives, in order, `0x20`, `0x2F`, a zero-extended byte freshly read at adjusted-base plus `0x04`, and `0x20`.

The global at `0x087FBBBC` is then read into the **saved anchor**, before the global at `0x60` is assigned `0x20`. The anchor is not reread following that assignment.

Each expanded publication group has this order:

1. Word publication of the publication payload at a scaled destination plus `0x54`.
2. Byte publication of the adjusted base at anchor plus `0x19`.
3. Byte publication of the saved anchor at adjusted-base plus `0x11`.
4. Word publication of the scaled destination at anchor plus `0x34`.

The first scaled destination is eight times the saved old word. Subsequent groups use eight times the adjusted value. These groups are statically expanded, not an established runtime loop.

**Scope limitation:** exact repetition count is not enumerated in this contract and remains **documented_only** here. No acceptance claim reduces the expanded region to one group or treats duplicate writes as removable.

### Final globals, variable shift, and rereads

After publication, the global at `0xA9` is captured into a 32-bit **saved fixed value**. A word read at publication-payload plus `0x74` supplies the **saved payload-linked word**.

The global at `0x3A` is assigned an expression formed by narrowing the adjusted value before multiplying by eight and adding `0x30`. Under low-bit arithmetic, the expression’s low byte is that of eight times the adjusted value plus `0x30`. This is not an assignment of constant `0x3A`; the global’s store width remains unresolved.

Words at tail-base plus `0x14` and adjusted-base plus `0x04` are saved. Byte writes then place the adjusted base at tail-base plus `0x0D`, `0x6C` at adjusted-base plus `0x08`, and the anchor at adjusted-base plus `0x0D` and `0x09`. Tail-base plus `0x54` receives the tail base as a word.

The global at `0x67` is assigned the saved fixed value explicitly narrowed to eight bits. A signed word read at tail-base plus `0x44` follows. Only then is `0x67` freshly read and assigned to global `0x21`. Neither global’s access width is established, and the copy is not specified as a direct copy of the original saved fixed value.

The signed loaded word supplies a byte at adjusted-base plus `0x10`, a word at anchor plus `0x54`, and a shifted-result byte at tail-base plus `0x01`. The shift uses the signed 32-bit word and the adjusted base’s low eight bits as count. Negative operands and counts at least 32 remain **documented_only**; no modulo-32 rule is established.

Anchor plus `0x54` is overwritten with the adjusted base. A word read at adjusted-base plus `0x40` precedes publication of the saved adjusted-base-`0x04` word at anchor plus `0x74`, the newly read word at adjusted-base plus `0x10`, and the tail base at anchor plus `0x44`.

Two further fresh global reads of `0x67` separately supply explicit byte stores at adjusted-base plus `0x09`. Between them, tail-base plus `0x09` receives the adjusted base’s low byte, adjusted-base plus `0x44` receives the saved fixed value as a word, and adjusted-base plus `0x54` is freshly read as a word. That last result is the **later destination base**.

### Final publications and terminal transfer

The saved payload-linked word’s low byte is published at later-destination-base plus `0x0D`. Tail-base plus `0x54` is restored from the saved tail-base-`0x14` word. A word read at adjusted-base plus `0x14` supplies a saved value before the payload-linked byte is published at tail-base plus `0x11`.

Adjusted-base plus `0x09` receives the later destination base’s low byte, and its `0x44` word receives the whole base.

A word read at tail-base plus `0x04` supplies the **final base**. A halfword `0x65` is stored at twice that base before its `0x64` word is read. Later-destination-base plus `0x11` receives the saved adjusted-base-`0x14` word’s low byte; its `0x54` word receives decimal `100`.

The final-base-`0x74` word is read before the tail-base-`0x64` word. Tail-base plus `0x09` then receives the final base’s low byte. The saved final-base-`0x64` word is published at saved-tail-base-`0x64` plus `0x54`.

The last direct store is a word whose address is twice the saved final-base-`0x64` word and whose value is the saved tail-base-`0x64` word.

The terminal invocation receives:

1. Literal `0x72`.
2. The saved final-base-`0x64` word.
3. The earlier saved tail-base-`0x14` word.
4. The later saved final-base-`0x74` word.

## Side effects

Computed accesses and fixed-global events may overlap. Earlier stores can therefore change later reads, including reads following clears. Distinct address derivations do not establish disjoint storage.

Explicit access widths constrain computed accesses; undeclared-global widths prevent complete byte-level overlap analysis. Endianness further affects overlap consequences.

No allocation, release, ownership transfer, reference counting, locking, atomicity, rollback, or reentrancy guarantee is established. Unknown dependencies are not assumed to preserve globals or to own or release memory.

Faults or nontermination can leave preceding effects in place. No explicit error-reporting or restoration path is represented. Device-visible ordering requires additional target information.

## Dependencies

| Opaque reference | Role | Unresolved properties |
|---|---|---|
| F-0710eaaaca42-D01 | Initial indirect invocation | Destination, ABI, outputs, effects, termination |
| F-0710eaaaca42-D02 | Ordered intervening invocation population | Individual identities, full schedule, secondary outputs, effects, ownership |
| F-0710eaaaca42-D03 | Terminal four-argument invocation | Destination, argument interpretation, effects, return behavior |

D02 denotes a population, not one established callee.

## Acceptance examples

These are conditional static requirements, not executed tests. Required preceding events, supported arithmetic, and accessible storage are assumed.

- **Explicit promotion:** if the unsigned-byte read at primary-base plus `0x05` yields `0xE7`, its associated word publication carries `0x000000E7`.
- **Converted global base:** if the absolute-`0x05` read and unsigned conversion produce `0x180`, the associated word read and replacement use `0x1D4`, assuming nonoverflowing address addition. A single underlying byte does not establish that conversion result.
- **Intervening small-field publication:** after replacement through fixed-global-base plus `0x54`, the small field must be word-published at small-base plus `0x34` before the primary-base-`0x74` read. If these accesses alias, that read observes the effects permitted by the earlier store and target layout.
- **Primary-linked overwrite:** if the base seed is `1`, the first primary-linked-`0x54` payload is `0x04000000`. The later payload is the independently saved shifted word, with the small-base-`0x74` expression and first self-publication intervening.
- **Global widths:** a saved value of `0x12345678` from `0xA9` constrains the captured 32-bit quantity, not an unconditional four-byte global load. Assigning its narrowed value `0x78` to `0x67` likewise does not establish a one-byte store.
- **Fresh global copy:** if the intervening tail-relative word read or overlapping memory effects change what `0x67` supplies, the assignment to `0x21` uses the fresh global value under the unresolved global types.
- **Adjusted expression:** adjusted value `2` gives low expression byte `0x40` for the assignment to `0x3A` under low-bit arithmetic; it does not establish the destination’s storage width.
- **Shift boundary:** signed operand `256` and count `4` produce narrowed byte `0x10` under conventional defined right-shift behavior. Negative operands or count `32` have no unconditional expected byte here.
- **Completion:** D03 receives the specified literal and saved values only after preceding events complete. Normal return remains conditional on D03 returning.

## Uncertainties

The following remain **documented_only**:

- Application meaning, object layouts, allocation boundaries, and completion reachability.
- Boundary accuracy, complete ABI, individual dependency identities, and secondary-result validity.
- Exhaustive intervening invocation schedule and exact expanded-publication count.
- Storage widths, signedness, extension behavior, and destination conversions of every undeclared fixed global listed above.
- Signed overflow, signed shifts, address overflow, and oversized shift behavior.
- Endianness, alignment, memory-map permissions, and device semantics.
- Precise machine ordering of reads combined within one expression.
- Dependency mutations, ownership actions, faults, and nontermination.

Textual, packed-field, and linked-state interpretations are inferred intent only.

## Evidence

Established claims derive from explicit interface widths, pointer-access widths, conversions, masks, saved-value dependencies, and represented ordering.

The unsigned-byte read at primary-base plus `0x05` has explicit width evidence. Named fixed globals do not acquire such evidence merely from assignment to a word-sized saved quantity or assignment of an eight-bit-narrowed expression.

The mutation ordering specifically includes the small-base-`0x34` word publication before the primary-base-`0x74` read, the primary-linked-`0x54` publication derived from the base seed shifted by 26 before the small-base-`0x74` expression, and its later overwrite between repeated self-publications.

Comments, prior prose, and reviewer feedback do not independently establish semantics. No dependency bodies, executable traces, global declarations, or target-specific arithmetic definitions were supplied. No runtime tests have been performed.
