<!-- Trusted provenance: F-f03f36ee9e5e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f03f36ee9e5e

## Purpose

F-f03f36ee9e5e performs ordered, mixed-width memory mutations, prepares part of a stack-relative argument area, and invokes an entry-selected target. If that invocation returns, it prepares five further words and invokes F-cdec09f5d432. If that invocation returns, execution reaches a represented terminal operation and invocation through its result.

Destructive preparation followed by dispatch is established by the supplied static representation. Application purpose, object types, ownership protocol, and successful completion are not established.

This contract separates established accesses and dependencies from inferred intent and **documented_only** unknowns. All later effects are conditional on execution reaching them. Two bytes are consumed before any visible assignment; their earlier values are not established. No runtime tests have been performed.

## Interface

### Explicit inputs and implicit context

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | No direct use in the supplied body |
| Second argument | 32 bits | No direct use in the supplied body |
| Third argument | Signed 32 bits | Displacement in the opening halfword destination |

The represented result is void. No status result or ordinary return path is established.

The body additionally depends on incoming machine state, identified here by behavioral roles:

- **Publication anchor:** an address also stored as a 32-bit payload.
- **Displacement anchor:** a base combined with the third argument and subsequently read at offset 0x44.
- **Initial active address:** the initial base for active-field accesses.
- **Entry payload:** a 32-bit value stored during the first selection.
- **Entry-selected target:** the first invocation target.
- **Two unresolved bytes:** values used before their visible assignments.

These roles do not establish additional formal parameters or their initialization mechanism. The complete ABI is **documented_only**.

### Widths, arithmetic, and prerequisites

Offsets are byte offsets. Byte, halfword, and word accesses have widths of 8, 16, and 32 bits respectively. Unqualified field observations below are word observations.

Arithmetic halfword observations are signed 16-bit values: 0xffff contributes −1 when promoted. Character observations are eight bits, but their signedness on promotion is unresolved. Character-derived addresses and halfword payloads therefore depend on that unresolved promotion.

Narrow stores carry the represented low byte or halfword. This does not establish portable behavior for every out-of-range signed conversion.

Address calculations use signed addition, multiplication, doubling, and left shifts. Universal modulo-2³² arithmetic is not established. Overflow, invalid signed shifts, address conversion, alignment, accessibility, and valid invocation targets are preconditions or **documented_only** limitations, not checked conditions. No bounds, null, permission, or alignment checks are represented.

## Behavior

### Access model and selection boundaries

The operation is not an immutable-entry-snapshot transformation. Later observations see memory at their represented access points. Captured values remain available independently of subsequent changes to their original sources, except where the saved value itself occupies storage that an overlapping write can change.

No separation is established among source cells, selected storage, linked storage, integer-derived destinations, and the argument area. Repeated observations must not be merged merely because their addresses agree. No atomicity, synchronization, or physical transaction granularity is established.

There are eighteen distinct fixed-data selection cells. **S1–S18** identify the addresses selected from them in order, not the cells themselves. Each boundary performs, in order:

- Self-publication of the indicated active address into its own 0x54 field.
- Observation of the next selection cell to capture the selected address.
- Publication of another reference to that cell’s word into the indicated recipient’s 0x44 field.
- Initialization of selected offset 0x14.

No explicit write separates the two selection-cell references. Earlier writes, including boundary self-publication, can affect selection through overlap.

| Selection | Self-publication base | Publication recipient |
|---|---|---|
| S1 | Initial active | Publication anchor |
| S2 | Initial active | First followed link |
| S3 | Initial active | Second followed link |
| S4 | Initial active | Captured initial-active 0x14 |
| S5, S6 | Initial active | Captured initial-active 0x44 |
| S7 | Initial active | Captured S6 0x14 |
| S8 | Initial active | Captured S7 0x14 |
| S9 | Initial active | Captured S8 0x14 |
| S10 | First successor active | Initial-active 0x44 captured during S9 |
| S11 | Second successor active | First-successor-active 0x44 captured during S10 |
| S12, S13, S14 | Second successor active | Character captured during the preceding selection |
| S15 | Third successor active | Second-successor-active 0x44 captured during S14 |
| S16 | Third successor active | Third-successor-active 0x44 captured during S15 |
| S17 | Fourth successor active | Third-successor-active 0x44 captured during S16 |
| S18 | Fifth successor active | Fourth-successor-active 0x44 captured during S17 |

S1 0x14 receives the opening displacement-anchor observation. Every subsequent selected 0x14 initially receives the preceding **handoff**. Replacements and overlap can change it immediately.

Each handoff is a fresh 0x44 observation at the end of its selection. Its base is the selected address except at S10, S14, and S15, as specified below. S18 instead enters the closing computation.

### Opening and S1–S5

The opening captures initial-active 0x14 as the **opening referent**. Publication-anchor 0x54 receives that anchor itself. Displacement-anchor plus the third argument receives the publication anchor’s low halfword. Only afterward is displacement-anchor 0x44 captured.

After the S1 boundary, initial-active 0x11 receives the low byte of twice the initial active address. Opening-referent 0x64, initial-active 0x44, and the signed halfword at initial-active plus S1 are captured, in that order. Opening-referent 0x15 then receives the opening displacement word’s byte. Only after that store is the captured active link followed through 0x14, yielding the **first followed link**.

S1 0x24 receives the entry payload. S1 0x74 is captured before S1 0x0d receives the captured opening-referent byte. Signed-halfword plus S1 receives the first followed link’s low halfword.

During S2, initial-active 0x11 first receives that earlier opening-referent byte. Active 0x44 and the halfword at active plus S2 are captured before opening-referent 0x15 receives the incoming handoff’s byte. Following the captured link through 0x14 then yields the **second followed link**. S2 0x24 receives the captured S1 0x74; S2 0x74 is independently captured. S2 0x14 is replaced with the opening referent. Initial-active 0x11 receives that referent’s byte, second-followed-link 0x09 receives S2’s byte, and signed-halfword plus S2 receives the second followed link’s halfword.

S3 writes the doubled-active byte before observing its signed halfword. Opening-referent 0x09 receives the incoming handoff’s byte. A fresh active 0x14 observation supplies the next publication recipient and halfword payload. A fresh opening-referent 0x64 observation supplies initial-active 0x11 before the halfword store.

S4 writes the doubled-active byte, then captures opening-referent 0x64, active 0x44, and its signed halfword. Opening-referent 0x54 receives the captured S2 0x74. S4 0x09 receives the active address’s byte; captured-link 0x0d receives S4’s byte. Signed-halfword plus S4 receives the captured link’s halfword.

S5 writes the S4 opening-referent byte to active 0x11, then captures active 0x44 and its signed halfword. S5 0x01 receives S5’s byte; S5 0x0d receives the captured S2 0x74 byte; captured-link 0x09 receives S5’s byte. Its halfword payload is that captured link.

### S6–S8 and retained storage

S6 writes the doubled-active byte and observes its signed halfword. Signed-value 0x44 yields the **first redirected base**. S6 0x14 is captured as the next publication recipient and halfword payload. A fresh opening-referent 0x64 observation supplies active 0x11 before the halfword store.

Immediately after S7 0x14 initialization, active 0x11 receives the doubled-active byte. This store precedes, in order, first-redirected-base 0x64, active 0x44, and signed-halfword observations. It can affect any of them through overlap.

The incoming handoff, used as an address, then receives the initial active address at 0x54. The captured redirected byte is written to active 0x11. Captured-active-link 0x44 receives S7. Signed-value 0x44 yields the **second redirected base**, and S7 0x14 supplies the next recipient and halfword payload. The redirected byte is written to active 0x11 again before the halfword store. Neither byte store is redundant under the aliasing model.

Immediately after S8 0x14 initialization, active 0x11 receives the doubled-active byte. Only afterward is the signed halfword observed, followed by signed-value 0x44, which establishes the **retained base**. The retained base is saved at argument-area offset 0x34.

S8 0x14 is captured. A fresh second-redirected-base 0x64 observation supplies active 0x11. Retained-base 0x04 and S8 0x74 are then captured; the latter is the **eighth carried word**. Active 0x54 temporarily receives the retained-base 0x04 word. The halfword destination is S8 plus twice the initial active address, with captured S8 0x14 as payload. The next boundary overwrites active 0x54 with self-publication.

Later retained-base references use the saved state available at that point. Overlap with argument-area offset 0x34 can change subsequent references.

### S9–S10: separate active and followed paths

S9 writes the captured retained-base 0x04 byte to active 0x11. It captures retained-base 0x64, active 0x44, and its signed halfword. Retained-base 0x54 receives the eighth carried word.

S9 0x14 yields the **first successor active address**, whose 0x0d receives S9’s byte. Signed-halfword plus S9 receives the earlier active-link halfword. The S9 handoff is captured before the next boundary.

S10 writes the preceding retained-base byte to first-successor-active 0x11, then captures that active address’s 0x44 and its signed halfword. Retained-base 0x54 again receives the eighth carried word.

S10 0x14 yields the **second successor active address**, whose 0x0d receives S10’s byte. Independently, incoming-handoff 0x14 yields the **split-path base**. The eighth carried word, used as an address, receives the retained base at 0x54.

The halfword destination is signed-value plus split-path base; its payload is the captured first-successor-active link. The outgoing handoff comes from split-path-base 0x44, not S10 0x44.

### S11–S13: character-dependent accesses

S11 writes the doubled second-successor-active byte. Its signed halfword plus incoming handoff identifies a character source. S11 0x1d receives the eighth carried word’s byte. Signed-value plus S11 receives the character converted to a halfword. The character also determines the next publication recipient.

S12 likewise writes the doubled-active byte before observing its signed halfword and character. A fresh retained-base 0x64 word is stored at signed-value 0x54. Character-derived 0x0d receives the incoming handoff’s byte. S12 0x64 is captured as the **middle carried word** before the character halfword store.

Immediately after S13 0x14 initialization, second-successor-active 0x11 receives the doubled-active byte. This precedes the retained-base 0x64, signed-halfword, and character observations, in that order. Overlap can therefore change their captured values and resulting destinations.

Retained-base 0x01 receives S13’s byte. S13 0x0d receives the incoming handoff’s byte, and S13 0x44 is replaced with the middle carried word. Character-derived 0x0d receives S13’s byte before the character halfword store. Only afterward is S13 0x44 reread for the handoff; intervening overlapping stores can change the replacement.

### S14–S17: further active transitions

S14 writes the earlier retained-base byte to active 0x11, then captures fresh retained-base 0x64, active 0x44, and its signed halfword. Incoming-handoff 0x74 yields the **carried-0x74 base**. A separate active 0x14 observation yields the **third successor active address**.

Captured-active-link 0x0d receives the carried-base byte. Signed-value plus carried base receives the captured active-link halfword. The handoff comes from carried-base 0x44.

S15 writes the preceding retained-base byte to third-successor-active 0x11 and captures active 0x44 and its signed halfword. Incoming-handoff 0x15 receives S15’s byte. S15 0x44 receives that handoff. Only then does captured-active-link 0x74 yield the **linked-0x74 base**, which supplies both the halfword destination base and outgoing 0x44 handoff. The halfword payload is the captured active link.

S16 writes the doubled-active byte, then captures retained-base 0x64, active 0x44, and its signed halfword. Signed-value 0x14 yields the **fourth successor active address**.

The captured active link receives self-publication at 0x54. S16 plus incoming handoff receives the middle carried word as a word store. S16 0x14 receives the captured active link. A fresh middle-carried-word 0x44 observation supplies S16 0x0d. Fourth-successor-active 0x0d and signed-value plus S16 receive the captured link’s narrow portions.

S17 writes the preceding retained-base byte to fourth-successor-active 0x11, then captures retained-base 0x64, active 0x44, and its signed halfword. Incoming-handoff 0x54 receives the fourth successor active address. The first unresolved byte is stored at active 0x11. Captured-active-link 0x44 receives S17.

Only afterward does signed-value 0x14 yield the **fifth successor active address**. Captured-active-link 0x54 receives itself. S17 plus incoming handoff receives the middle carried word; S17 0x14 receives the captured link. A fresh middle-carried-word 0x44 observation supplies S17 0x0d. Fifth-successor-active 0x0d and the halfword destination receive the captured link’s narrow portions.

After the S18 boundary, the first unresolved byte is visibly assigned from S17’s retained-base 0x64 capture and written to fifth-successor-active 0x11. That assignment does not determine its earlier use.

### S18 and closing captures

S18 captures fifth-successor-active 0x44 and a signed halfword called the **factor**. Factor 0x54 receives self-value stores before, and between, two stores of the active address’s byte at S18 0x11.

Factor multiplied by the captured active link determines a 0x44 source. Its word is followed through 0x74 and then 0x14 to identify a destination. That destination’s old 0x14 word is captured before replacement with twice the active address.

Factor multiplied by old-word 0x14 forms the **first product**. After factor self-publication, S18 0x11 receives the old word’s byte. First-product 0x74 supplies the **retained product word**, saved at argument-area offset 0x108. Factor multiplied by the first product forms the **second product**. Another factor self-publication precedes the second unresolved byte’s store at S18 0x11.

Closing observations and mutations retain this order:

1. Second-product 0x44 and retained-base 0x74 are captured.
2. S18 0x64 receives the saved retained product word; S18 0x09 receives the first captured 0x44 byte.
3. Second-product 0x44 is observed again and supplies second-product 0x11.
4. Following factor self-publication, the second unresolved byte is assigned from the saved retained product word and stored at twice-second-product 0x11.
5. Factor-times-second-product 0x44 is captured, followed by retained-base 0x14 as the closing destination.
6. Retained-product-word 0x01 receives the captured retained-base 0x74 byte.
7. Closing-destination 0x11 receives the factor-product observation’s byte; its 0x09 receives twice the second product’s byte.

The two second-product observations can differ. Both late captures precede the retained-product-word byte store. Saved argument-area words remain subject to overlapping writes.

### Argument preparation and invocation

Under noninterfering storage and supported arithmetic, preparation establishes:

| Argument-area offsets | Word value |
|---|---|
| 0x10, 0x24 | Retained base shifted left four bits |
| 0x34, 0x4c, 0x13c, 0x188 | Retained base shifted left eight bits |
| 0x108, 0x128, 0x17c | Retained product word |
| 0x60, 0xa8, 0xe4, 0x1b8 | Zero |

Offset 0x10 is prepared before replacing offset 0x34 with its eight-bit-shift result. Zero stores occur at 0xe4, 0x60, 0xa8, and 0x1b8. Subsequent copies target 0x24, 0x4c, 0x128, 0x13c, 0x17c, and 0x188, in that order. Unlisted words are not initialized by this preparation.

F-opaque-01 receives the area base pointer and two zeros. No returned value is visibly consumed.

If it returns, stores occur at offsets 0x08, 0x19c, 0x100, 0x120, and 0x140, in that order. They receive zero except 0x19c, which receives 0x08400e1c.

F-cdec09f5d432 then receives a freshly observed fixed-data word and three zeros. If it returns, F-opaque-03 represents a terminal operation with immediate zero and a fixed instruction-location operand. Its result supplies F-opaque-02’s invocation target; no explicit arguments are represented.

## Side effects

Established effects are ordered memory mutations, partial argument-area preparation, and dependency interactions.

No allocation, release, ownership transfer, reference counting, locking, rollback, or synchronized publication is established. Address publication does not imply ownership transfer.

Dependencies are not known to preserve globals, source cells, linked storage, the argument area, or implicit machine state. Passing stack-relative storage establishes neither permission to retain it nor protection against retention or release.

Invalid accesses, unsupported arithmetic, invalid targets, or nonreturning dependencies can prevent later effects. No error result, recovery path, or restoration is represented.

## Dependencies

| Opaque ID | Established interaction | Unresolved properties |
|---|---|---|
| F-opaque-01 | Entry-selected invocation with area pointer and two zeros | ABI, effects, ownership, preservation, return |
| F-cdec09f5d432 | Invocation with fresh fixed-data word and three zeros | Argument meaning, ABI, effects, ownership, return |
| F-opaque-03 | Represented terminal operation producing a target | Concrete semantics and target formation |
| F-opaque-02 | Invocation through that result | Reachability, ABI, effects, terminal mechanism |

The eighteen selection cells and later call-argument cell are data dependencies. Their contents, accessibility, mutability, lifetimes, and separation from destinations are unresolved.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Opening width:** Anchor low halfword 0xabcd produces a 16-bit store of that pattern, not a word replacement.
2. **Selection timing:** If boundary self-publication overlaps a selection cell, selection follows that write rather than using an entry snapshot.
3. **S7 overlap:** If initial-active 0x11 overlaps first-redirected-base 0x64, S7 captures that word after the doubled-active byte store. The same ordering applies to its active-link and halfword observations.
4. **S8 overlap:** If initial-active 0x11 overlaps S8’s halfword source, the doubled-active byte store precedes the halfword observation and the dependent retained-base observation.
5. **S13 overlap:** If second-successor-active 0x11 overlaps retained-base 0x64 or S13’s halfword source, the doubled-active byte store precedes those captures and the character observation.
6. **Signed boundary:** Halfword 0xffff contributes −1. Character 0xff has no established equivalent promotion.
7. **Independent paths:** S10 selected 0x14 determines the next active address; incoming-handoff 0x14 independently determines the split-path base.
8. **Replacement sensitivity:** A narrow store overlapping S13 0x44 after its replacement can change the outgoing handoff.
9. **Saved observations:** Replacing the product-chain destination’s 0x14 does not change its already captured old word.
10. **Closing overlap:** The retained-product-word byte store cannot redirect the already captured closing destination.
11. **Partial preparation:** Retained base 0x1234 yields 0x12340 and 0x123400 under supported arithmetic and noninterference. Unlisted area words need not be zero.
12. **Unresolved bytes:** Neither earlier use is specified as zero or as its later assigned value.
13. **Invocation boundary:** A nonreturning F-opaque-01 prevents all five post-call stores and later invocations. If it returns after changing the later argument-source cell, the second invocation uses that cell’s subsequently observed contents.
14. **Failure:** Earlier completed writes have no represented rollback requirement.

## Uncertainties

The following remain **documented_only**:

- Application purpose, complete ABI, and origins of implicit state.
- Consequences of both pre-assignment byte uses.
- Storage identity, extent, lifetime, overlap, byte order, and alignment.
- Character promotion, arithmetic overflow, shifts, and address conversions.
- Selection-cell contents and accessibility.
- Unassigned argument-area contents and complete area extent.
- Dependency ownership, retention, release, preservation, errors, and return behavior.
- Concrete terminal-operation semantics.
- Whether the unusual address computations faithfully describe executable behavior.

Coordinated mutation followed by dispatch is inferred intent, not an additional guarantee.

## Evidence

The supplied static body supports the represented access widths, eighteen ordered selections, boundary publications, active-address transitions, replacement-sensitive handoffs, product-dependent closure, partial argument preparation, and invocation order.

In particular, it explicitly places the doubled-active byte stores immediately after selected 0x14 initialization at S7, S8, and S13, before the subsequent observations identified above.

Comments, the prior draft, and reviewer feedback are not independent evidence of reachability, storage separation, or dependency semantics. No dependency implementations, external ABI contract, execution traces, or dynamic observations were supplied. No runtime tests have been performed.
