<!-- Trusted provenance: F-ff29a5d0e42a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff29a5d0e42a

## Purpose

F-ff29a5d0e42a transforms address-connected memory, exposes one argument’s storage to an entry-supplied callable, invokes a further dependency, and finishes with an indirect invocation. The supplied static representation establishes neither an application-level result nor an ordinary return path.

The contract is **order-sensitive**, not merely a description of final memory. Narrow writes can modify later word observations; published words can alter subsequent address selection; temporary values can influence later effects before being overwritten. Equivalence therefore requires preserving these dependencies, including fresh observations and intermediate writes where overlap matters.

Maintenance of linked state is an inferred interpretation only. No validated object model, ownership protocol, or recovery mechanism is established.

Two byte payloads lack preceding visible definitions. Their values and execution consequences remain unresolved. Behavior described beyond those uses is conditional on execution continuing.

## Interface

### Explicit inputs

| Position | Represented width | Established role |
|---|---:|---|
| 1 | 32 bits | Address base for a word observation at byte offset 0x64. |
| 2 | 32 bits | Incoming value has no direct represented use. |
| 3 | 32 bits | Address contribution to the initial halfword write. |
| 4 | 8 bits | First byte payload at the entry destination. |
| 5 | 32 bits | Address of its storage is exposed to F-opaque-01. |
| 6 | 32 bits | Incoming value has no direct represented use. |
| 7 | 32 bits | Its represented storage is cleared after F-opaque-01 returns. |

Additional entry state supplies three 32-bit address-like values: the **initial base**, **link base**, and **entry destination**. It also supplies the callable identified here as F-opaque-01. Their ABI origins are **documented_only**.

Twelve external word sources participate in successive publication phases. A separate external word source supplies the first argument of F-cdec09f5d432. Their storage independence, stability, and alias relationships are not established.

### Widths, addressing, and preconditions

Offsets below are byte offsets. Byte, halfword, and word effects have widths of 8, 16, and 32 bits. No enclosing allocation size follows from any field offset.

Halfword observations used as numeric address contributions are signed, with range −32768 through 32767. Character observations undergo promotion; plain-character signedness is unresolved. An observed character with bits 0xFF therefore does not determine a unique numeric address contribution.

Address arithmetic and products use represented 32-bit integer operands; wider products are not established. Multiplication by two and left shift by one are distinct represented operations. Left shifts by four and eight also occur. Neither universal wraparound arithmetic nor validity of every shift is established.

Every reached access requires appropriate storage, permissions, lifetime, alignment, and address interpretation. Every invocation requires a suitable target and calling convention. There is no explicit validation, error result, or rollback. Invalid arithmetic, inaccessible memory, or a dependency that does not return can prevent later effects without undoing earlier ones.

## Behavior

### Publication protocol and retained state

The transformation contains twelve ordered **publication phases**. This numbering describes behavioral boundaries; it does not imply an implementation loop.

Each phase has a selector, an incoming link, a replacement base, and an opening destination. The current destination begins as the entry destination and changes only at the redirections specified below.

The common opening has four ordered obligations:

1. An observation of that phase’s external source determines its selector.
2. Another observation of the same source supplies a word at replacement-base offset 0x44.
3. The incoming link is published at selector offset 0x14.
4. The opening byte is written at opening-destination offset 0x11.

These distinguish represented value observations, not guaranteed physical transactions. There is no intervening represented write between the two source observations, but equality is not guaranteed by this evidence.

Replacement-base selection follows this progression:

| Phase | Replacement base |
|---|---|
| 1 | Initial base |
| 2–3 | Previous phase’s retained selector-offset-0x14 observation |
| 4–5 | Previous phase’s saved word |
| 6–8 | Previous phase’s promoted character |
| 9–12 | Previous phase’s saved word |

A **saved word** is an observation at opening-destination offset 0x44. A **signed contribution** is a signed halfword observed at opening destination plus selector. Where both occur, their order is specified below.

Phases 1–11 finish by freshly observing outgoing-base offset 0x44, then writing the current destination’s self-reference at its offset 0x54. The fresh word becomes the next incoming link. The observation precedes the self-reference even when those locations overlap.

The outgoing base is normally the selector. Phases 4, 8, and 9 instead use their further base.

### Establishment and first two phases

Before publication begins, a halfword conversion of the initial base is written at argument 3 plus link base. Link-base offset 0x44 is observed afterward to establish the first incoming link. Entry-destination offset 0x54 then receives the entry destination.

Phase 1’s opening byte is argument 4. Its subsequent observations are ordered: argument-1 offset 0x64, saved word, signed contribution. The argument-relative word is retained for two later byte replacements at destination offset 0x11; there is no second argument-relative observation.

Before the first replacement, incoming-link offset 0x54 receives the opening destination. Between the two replacements, saved-word offset 0x44 receives the selector, a word at signed-contribution offset 0x44 establishes the **preliminary link**, and a fresh selector-offset-0x14 observation establishes the halfword payload and phase 2’s replacement base. The second replacement precedes the halfword write. That write targets signed contribution plus selector.

Phase 2 opens with a byte derived from entry destination shifted left by one. Its signed contribution leads to a word observation at contribution offset 0x44, establishing the **anchor** in represented frame storage at offset 0x34. This placement precedes the selector-offset-0x14 observation retained for phase 3’s replacement base and the current halfword payload.

After that selector observation, preliminary-link offset 0x64 is freshly observed and supplies another destination byte. Anchor offset 0x04 is then observed, followed by selector offset 0x74, which establishes the **companion**. The anchor-offset-0x04 word temporarily occupies destination offset 0x54.

The phase 2 halfword destination is **twice entry destination plus selector**, using multiplication by two. It does not use the signed contribution. The closing self-reference replaces the temporary offset-0x54 word.

### Destination and reciprocal-link phases

Phase 3 opens with a byte derived from the retained anchor-offset-0x04 word. It freshly observes anchor offset 0x64, then the saved word, then the signed contribution. Anchor offset 0x54 receives companion before a fresh selector-offset-0x14 observation redirects the current destination. A selector-derived byte is written at redirected-destination offset 0x0D. The halfword write targets signed contribution plus selector and carries the saved word.

Phase 4 opens using the retained phase 3 anchor-offset-0x64 observation. Saved word precedes signed contribution. As in phase 3, companion is published at anchor offset 0x54 before selector offset 0x14 redirects the destination, and the redirected destination receives a selector-derived byte at offset 0x0D.

Only after that byte is the **further base** observed at incoming-link offset 0x14. Companion offset 0x54 then receives anchor. The halfword targets signed contribution plus further base and carries the saved word. Closing link acquisition is relative to further base, not the original selector.

### Character-dependent phases

Phases 5–7 open with bytes derived from opening destination shifted left by one. Each observes its signed contribution before observing a character at signed contribution plus incoming link. The promoted character supplies both the current halfword payload and the next phase’s replacement base. Each halfword targets signed contribution plus selector.

Their additional requirements differ:

- **Phase 5:** after the character observation, selector offset 0x1D receives a companion-derived byte.
- **Phase 6:** after the character observation, a fresh anchor-offset-0x64 word is published at signed-contribution offset 0x54. The character-valued base receives an incoming-link-derived byte at offset 0x0D. Selector offset 0x64 is then observed to establish the retained **replacement payload**.
- **Phase 7:** a fresh anchor-offset-0x64 observation occurs immediately after the opening byte, before the signed contribution and character observations. After the character observation, the ordered effects are a selector-derived byte at anchor offset 0x01, an incoming-link-derived byte at selector offset 0x0D, replacement payload at selector offset 0x44, and a selector-derived byte at character-valued-base offset 0x0D.

These effects precede their respective halfword writes. Phase 6’s anchor observation is independent of phase 7’s retained refresh.

Phase 7’s closing selector-offset-0x44 observation is fresh. It cannot unconditionally be replaced by the earlier replacement payload: intervening byte or halfword writes may overlap that word.

### Further-base and replacement-payload phases

Phase 8 opens using phase 7’s retained anchor refresh, then refreshes anchor offset 0x64 again. Saved word precedes signed contribution. Incoming-link offset 0x74 establishes further base before opening-destination offset 0x14 redirects the destination. Saved-word offset 0x0D receives a further-base-derived byte. The halfword targets signed contribution plus further base and carries saved word.

Phase 9 opens using phase 8’s retained anchor refresh; it does not refresh that source. Saved word precedes signed contribution. An incoming-link-offset-0x15 byte receives selector, and selector offset 0x44 receives incoming link. Only then is further base observed at saved-word offset 0x74. The halfword again targets signed contribution plus further base and carries saved word.

Phase 10 opens with an opening-destination-shifted-left-one byte. Phase 11 opens using phase 10’s retained anchor-offset-0x64 observation. Both refresh anchor offset 0x64 immediately after their opening bytes, then observe saved word before signed contribution.

Both redirect through signed-contribution offset 0x14. Before phase 11’s redirection, however, incoming-link offset 0x54 receives opening destination, opening-destination offset 0x11 receives the first unresolved byte payload, and saved-word offset 0x44 receives selector, in that order.

After redirection, both phases require this ordering:

- Saved-word offset 0x54 receives saved word.
- A word at selector plus incoming link receives replacement payload.
- Selector offset 0x14 receives saved word.
- A fresh replacement-payload-offset-0x44 observation supplies a byte at selector offset 0x0D.
- Redirected-destination offset 0x0D receives a saved-word-derived byte.
- A halfword conversion of saved word is written at signed contribution plus selector.

### Multiplicative closure

Phase 12 opens using phase 11’s retained anchor refresh. Saved word precedes the signed halfword observation at opening destination plus selector. Here that signed value is the **factor**.

Five word publications of factor occur at factor offset 0x54. Their intermediate positions matter:

- The first two each precede a destination-derived byte at selector offset 0x11.
- The third precedes a chain-child-derived byte there.
- The fourth precedes the second unresolved byte payload there.
- The fifth follows the fresh later-base word-to-byte effect described below.

After the first two publication-and-byte pairs, factor multiplied by saved word initiates dependent word observations through offsets 0x44, 0x74, and 0x14. The final observation establishes the **chain origin**.

Chain-origin offset 0x14 is observed to obtain the **chain child** before that origin field receives destination shifted left by one. A fresh chain-child-offset-0x14 observation occurs after the replacement. Multiplying that observation by factor establishes the **intermediate product**.

After the third factor publication and its child-derived byte, intermediate-product offset 0x74 is observed to obtain the **product result**. The result is immediately placed in represented frame storage at offset 0x108.

The **later product base** is factor multiplied by intermediate product, not factor multiplied by product result. Its formation precedes the fourth factor publication and unresolved byte.

Later-product-base offset 0x44 is observed before anchor offset 0x74. Product result is published at selector offset 0x64, followed by a byte derived from the retained later-base word at selector offset 0x09. A fresh later-product-base-offset-0x44 observation then supplies a byte at that base’s offset 0x11.

After the fifth factor publication, a product-result-derived byte is written at twice later product base plus 0x11. This doubling is multiplication. Its later-defined payload does not resolve the earlier unresolved byte.

A word at factor multiplied by later product base, plus 0x44, establishes the **selected word**. Anchor offset 0x14 is then observed to select the final destination.

The final addressed writes, in order, are:

- An anchor-offset-0x74-derived byte at product-result offset 0x01.
- A selected-word-derived byte at final-destination offset 0x11.
- A byte derived from twice later product base at final-destination offset 0x09.

### Frame exposure and invocation boundary

Frame offsets describe represented 32-bit storage relationships, not a proven physical stack layout.

Before F-opaque-01, anchor shifted left by four is placed at offset 0x10; anchor shifted left by eight replaces the earlier anchor placement at offset 0x34. Zero placements follow at offsets 0xE4, 0x60, 0xA8, and 0x1B8, in that order.

Subsequent placements occur in this order:

| Offset | Value |
|---:|---|
| 0x24 | Retained shifted-by-four value |
| 0x4C | Retained shifted-by-eight value |
| 0x128 | Product result |
| 0x13C | Shifted-by-eight value |
| 0x17C | Product result |
| 0x188 | Shifted-by-eight value |

F-opaque-01 receives argument-5 storage’s address and two zeros. Its result is unused.

If it returns, argument-7 storage is cleared. Frame offset 0x19C receives token 0x08400E1C, followed by zeros at offsets 0x100, 0x120, and 0x140.

Only afterward is the separate external argument source observed. F-cdec09f5d432 receives that word and three zeros.

If execution continues, F-opaque-02 receives zero and token 0x08401154. Its supplied target is invoked as F-opaque-03 without represented explicit arguments. No ordinary return continuation is established.

## Side effects

Effects include overlapping byte, halfword, and word modifications; argument-storage modification and exposure; represented frame placements; dependency invocations; and terminal indirect control transfer.

No storage separation is guaranteed. Numeric values used as addresses need not designate distinct objects. Retained observations must remain distinct from fresh observations, but no physical load count, volatility, atomicity, or synchronization guarantee follows.

Passing argument-5 storage’s address establishes exposure, not ownership transfer, retention permission, release authority, or accessible extent.

No local allocation, release, rollback, or restoration protocol is represented. Dependencies may independently modify globals or reachable memory. An unused result does not establish preservation. Frame placements likewise do not prove isolation from addressed memory or dependencies.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-opaque-01 | Entry-supplied callable receiving argument-5 storage’s address and two zeros. |
| F-cdec09f5d432 | Receives the late external word observation and three zeros. |
| F-opaque-02 | Receives zero and a fixed token; supplies the terminal target. |
| F-opaque-03 | Invocation of that target without represented explicit arguments. |

Implementations, error behavior, return behavior, ownership, memory preservation, and nonlocal control effects are **documented_only**. F-opaque-02 is not established as an ordinary application API. Neither fixed token is established as a dereferenceable address or application-level identifier.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Initial overlap:** If the initial halfword write overlaps link-base offset 0x44, the incoming-link observation uses storage after that write.
- **Changing source:** If a phase’s external observations differ, the first selects addresses while the second supplies the offset-0x44 publication.
- **Signed contribution:** Halfword bits 0xFFFF contribute −1, not 65535. Address usability remains separate.
- **Phase 2 exception:** Its halfword address depends on twice entry destination, not the signed contribution used to locate anchor.
- **Retained input:** Phase 1’s two argument-derived replacement bytes use one argument-relative word observation.
- **Overlap-sensitive redirection:** Phase 4’s redirected byte can affect its later further-base observation if storage overlaps.
- **Fresh closing link:** Phase 7’s outgoing link reflects any overlapping intervening narrow writes, not necessarily replacement payload.
- **Chain overlap:** Replacing chain-origin offset 0x14 can affect the subsequent child-field observation; the intermediate product uses that later observation.
- **Distinct products:** Product result cannot substitute for intermediate product in forming later product base.
- **Unresolved bytes:** Neither unresolved payload may be assigned zero or retroactively supplied from a later definition without additional evidence.
- **Repeated effects:** Keeping only the final factor publication omits four represented intermediate writes and their overlap-sensitive consequences.
- **Dependency mutation:** If F-opaque-01 changes the late external source and returns, the subsequent observation supplies F-cdec09f5d432’s argument.
- **Interrupted continuation:** A nonreturning dependency prevents later effects; earlier effects are not rolled back.

## Uncertainties

The following remain **documented_only**:

- Complete ABI and origins of additional entry state.
- Application types, object bounds, structural invariants, and success criteria.
- Both unresolved byte values and the consequences of using them.
- Storage validity, separation, alignment, permissions, lifetime, and accessible extent.
- Character signedness, byte order, narrowing semantics, signed overflow, shift validity, and integer-to-address conversion.
- Physical frame layout and accessibility of represented frame storage through other addresses.
- Dependency implementations, ownership rules, preservation guarantees, failures, and return behavior.
- Physical access counts, concurrency visibility, fault delivery, and terminal execution semantics.

Continuation beyond unresolved-value uses is not evidence of a valid executable path. Annotations about omitted unreachable regions or overlapping external symbols establish neither extra reachable behavior nor a complete alias map.

## Evidence

**Established static evidence:** access widths; signed-halfword contributions; represented ordering; fresh versus retained observations; destination changes; character- and product-dependent addressing; intermediate and repeated writes; frame relationships; argument-storage exposure; dependency ordering; terminal indirect invocation.

**Inferred intent:** manipulation of linked or otherwise address-connected state. No application-level meaning is established.

**Unknown:** runtime validity, dependency effects, unresolved-value consequences, physical execution details, and successful end-to-end operation.

This contract is derived solely from the supplied static representation. Comments and the prior draft are not independent behavioral evidence. No runtime tests, execution traces, or independent implementation validation have been performed.
