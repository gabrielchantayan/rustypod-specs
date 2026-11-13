<!-- Trusted provenance: F-cbfec12d9f42; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbfec12d9f42

## Purpose

F-cbfec12d9f42 performs an ordered sequence of memory publications and address-dependent observations, prepares invocation storage, and transfers control to an incoming callable. If that call returns, it prepares further storage and calls F-cdec09f5d432. Return from that dependency leads to an exceptional terminal operation rather than an established ordinary return.

The contract concerns observable memory effects and control transfer. No application-level operation, container type, or ownership protocol is established.

Mutations are nontransactional. An earlier publication is not contingent on later completion. Matching only the final memory contents is insufficient where intermediate writes influence subsequent reads.

## Interface

### Explicit inputs

| Position | Represented width | Established role |
|---|---:|---|
| First | Signed 32 bits | Address-bearing value, stored value, and arithmetic operand |
| Second | 32 bits | No direct consumption established |
| Third | 32 bits | No direct consumption established |
| Fourth | 8 bits | Initial destination-byte value |
| Fifth | 32 bits | Its invocation-storage address is exposed to the incoming callable |
| Sixth | 32 bits | No direct consumption established |
| Seventh | 32 bits | Its invocation storage is zeroed if the incoming call returns |

“No direct consumption” does not establish isolation from dependency accesses. In particular, the accessible extent around the fifth input’s exposed storage is unknown.

No ordinary result value is established.

### Implicit inputs and terminology

The operation also depends on an incoming **context address**, **destination address**, and **callable target**.

Ten embedded word sources are distinguished below by encounter order. Each supplies an address-bearing observation and a separate observation for publication. They are not established as an array, disjoint objects, immutable constants, or callable targets. A separate embedded source is observed only after the incoming call returns.

The following descriptive roles identify retained values, not additional inputs:

- **Current destination:** the address receiving destination-relative publications at that point.
- **Propagated word:** a retained word carried between portions of the mutation sequence.
- **Saved routing word:** the fourth source’s offset-0x64 observation, retained for later use.
- **Routing halfword:** a signed 16-bit observation used in address calculations.

Invocation-local storage is also used. One byte is consumed without an established preceding assignment.

### Widths, layout, and preconditions

All offsets are in bytes. Words are 32 bits; halfwords are 16 bits; bytes are 8 bits.

| Address-bearing input | Directly accessed offsets |
|---|---|
| First input | Byte write at 0x01; word reads at 0x04, 0x14, 0x64, and 0x74; word writes at 0x54 |
| Context | Word reads at 0x14, 0x44, and 0x74 |
| Destination | Byte publications at 0x11; word publications at 0x54 |

Indirect accesses additionally use word fields at 0x14, 0x44, 0x54, 0x64, and 0x74, and byte fields at 0x01, 0x09, 0x0D, 0x11, 0x15, and 0x1D. Some addresses are sums, doubled values, or products rather than object-relative fields.

No complete object extent, alignment requirement, memory map, or disjointness guarantee is supplied. There is no represented validation phase. Continuation requires each reached access and arithmetic operation to permit it; safe execution for arbitrary inputs is not promised.

## Behavior

### Interpretation and ordering

A narrow publication retains only the low eight or sixteen bits of its value.

Routing halfwords are signed. Pattern 0xFFFF contributes −1 when widened for arithmetic. Plain-byte widening is unresolved: a high-bit-set byte used numerically may be sign-extended or zero-extended. Its directly published eight-bit pattern is unaffected by that uncertainty.

Address calculations include signed 32-bit addition, multiplication, doubling, and left shifts. A universal wrapping-address interpretation is not established. Overflow, invalid shifts, and the validity of integer-derived addresses remain **documented_only**.

The ordering requirements below apply to the represented continuation. They do not establish defined execution after an invalid access or consumption of indeterminate state.

Aliasing is material throughout:

- A byte or halfword publication can alter a later overlapping word observation.
- Retained observations survive subsequent changes to their original locations.
- Separate reads cannot be replaced by one snapshot without an applicable equivalence argument.
- Repeated writes cannot be removed merely because their values match.

These requirements do not imply atomicity, volatile access, hardware transaction boundaries, or concurrency guarantees.

### Entry publications and retained context

The context-offset-0x14 word is retained before the fourth input is published at destination offset 0x11. The first-input-offset-0x04 word is observed after that byte publication. The context-offset-0x74 word is then retained separately.

Destination offset 0x54 receives the first-input-offset-0x04 observation before receiving the destination address itself. Between these publications:

- A halfword containing the retained context-offset-0x14 value’s low sixteen bits is written at the context address plus twice the destination address.
- Context offset 0x44 is observed after that halfword write.

The first source’s address observation is retained separately from its publication observation. The publication observation goes to offset 0x44 through the retained context-offset-0x14 address. The retained source address receives the context-offset-0x44 observation at offset 0x14.

Destination offset 0x11 subsequently receives the first-input-offset-0x04 observation’s low byte. Neither the initial fourth-input byte nor the initial offset-0x54 word is dispensable.

The retained context-offset-0x74 word has three later roles:

- Both first-input-offset-0x54 writes use it, without rereading the context.
- Its address receives the first input at offset 0x54.
- Its low byte is published at the third source’s offset 0x1D.

It is distinct from the saved routing word obtained from the fourth source.

### Propagation structure

The ten source portions form a fixed progression. No retry, termination condition, or alternate successful path is represented.

Each portion retains its source address separately from the source observation used for its associated offset-0x44 publication. The publication target changes with the progression: it is selected through retained context state, destination-offset-0x44 observations, or widened byte-derived addresses. The ninth portion also repeats a publication using its already retained source address.

Each retained source address receives an offset-0x14 publication carrying the preceding propagated word. This is not always that field’s final value: the eighth and ninth portions overwrite it later.

The current destination changes through:

1. The first source’s offset-0x14 observation.
2. The second source’s offset-0x14 observation.
3. The destination’s own offset-0x14 observation during the sixth portion.
4. Signed-halfword-derived offset-0x14 observations during the eighth and ninth portions.

Destination-relative effects after a replacement use the replacement address. Self-address publications at destination offset 0x54 recur throughout, including when the same value was already present.

### Fresh first-input observations

First-input offset 0x64 is observed six times. Their timing and consumers differ:

| Observation | Timing and consumers |
|---|---|
| First | After the entry destination-byte replacement; supplies the second portion’s destination byte |
| Second | In the fourth portion; supplies a full-word publication at offset 0x54 through a signed-halfword-derived address |
| Third | In the fifth portion, after its doubled-destination byte publication; supplies the sixth portion’s destination byte |
| Fourth | In the sixth portion, after that destination-byte publication; supplies the seventh portion’s destination byte |
| Fifth | In the eighth portion, after its doubled-destination byte publication; supplies the ninth portion’s first destination byte |
| Sixth | In the ninth portion, after that first byte publication; supplies another ninth-portion destination byte and the tenth portion’s destination byte |

Entry-time caching would discard possible effects of intervening overlapping writes.

Other destination-offset-0x11 publications use the low byte of twice the current destination. They occur in the third, fourth, fifth, and eighth portions and are not interchangeable with first-input-derived bytes.

### Routing distinctions across the first seven portions

The first nine portions observe signed halfwords at the sum of the current destination and retained source address. The resulting halfword-dependent writes are not one uniform operation.

The first portion combines its routing halfword with the retained first-source address. Its dependent halfword value comes from a destination-offset-0x44 observation. Beforeward, first-input offset 0x54 receives the retained context word, the replacement destination is observed through first-source offset 0x14, and that destination’s offset 0x0D receives the source address’s low byte. The first source’s offset 0x44 is observed after the dependent halfword write.

The second portion similarly publishes a source-address byte through its replacement destination. Its dependent routing address, however, comes through the preceding propagated word’s offset 0x14, not directly from the second source. That observation follows the byte publication. The retained context word’s offset-0x54 publication of the first input precedes the dependent halfword write. Propagation continues from the routing address’s offset 0x44.

The third through fifth portions obtain a byte through the sum of their signed routing halfword and preceding propagated word. Its widened value supplies both a halfword value and subsequent address selection. Consequently, unresolved byte signedness affects more than the stored byte pattern.

Their distinct obligations are:

- **Third:** The third source receives the retained context word’s low byte at offset 0x1D before the byte-derived halfword publication.
- **Fourth:** The second fresh first-input-offset-0x64 observation is published as a word at routing-halfword offset 0x54. The byte-derived address receives the preceding propagated word’s low byte at offset 0x0D. The fourth source’s offset-0x64 word is saved before the dependent halfword publication.
- **Fifth:** First-input offset 0x01 receives the fifth source address’s low byte. The fifth source receives the preceding propagated word’s low byte at offset 0x0D and the saved routing word at offset 0x44. The byte-derived address receives the fifth source address’s low byte at offset 0x0D.

The fifth source’s offset 0x44 is reread after the intervening byte and halfword writes. Substituting the saved routing word would lose possible alias effects.

The sixth portion obtains a routing address through the preceding propagated word’s offset 0x74. It then replaces the destination through the previous destination’s offset 0x14. The earlier destination-offset-0x44 observation selects an offset-0x0D byte target and supplies the dependent halfword value; the routing address supplies that byte’s value and contributes to the halfword target. Propagation continues from the routing address’s offset 0x44.

In the seventh portion, the seventh source address’s low byte is published at offset 0x15 through the retained propagated-word address. The seventh source’s offset 0x44 is overwritten before a different routing address is obtained through the retained destination-offset-0x44 word’s offset 0x74. That address governs the dependent halfword target and subsequent offset-0x44 observation.

### Eighth- and ninth-portion redirection

Both portions retain a destination-offset-0x44 observation and obtain a replacement destination through the signed routing halfword’s offset 0x14.

Each then requires:

- A self-address word publication at offset 0x54 through the retained destination-offset-0x44 address.
- Publication of the saved routing word at the sum of the retained source address and preceding propagated word.
- Replacement of the source’s earlier offset-0x14 publication with the retained destination-offset-0x44 word.
- A fresh word observation through the saved routing word’s offset 0x44, whose low byte goes to source offset 0x0D.
- Publication of the retained destination-offset-0x44 word’s low byte at replacement-destination offset 0x0D.
- A dependent halfword publication using the retained source address and routing halfword, followed by a fresh source-offset-0x44 observation and replacement-destination self-publication.

The two saved-routing-word-offset-0x44 observations are separate. Intervening mutations can distinguish them.

The ninth portion has additional effects before replacement: the preceding propagated address receives the then-current destination at offset 0x54; the sixth fresh first-input-offset-0x64 observation supplies a second destination byte; and the retained destination-offset-0x44 address receives another offset-0x44 publication using the retained ninth-source address. These effects cannot be collapsed into the earlier source observation.

### Product-selected continuation

The tenth portion retains a signed routing halfword that is used both as an address and as a multiplier. Before product-selected traversal, that halfword-derived address receives two self-address publications at offset 0x54, alternating with two tenth-source-offset-0x11 publications of the current destination’s low byte.

The first product combines the signed halfword with the retained destination-offset-0x44 word. Its offset-0x44 observation leads through offset 0x74 and then offset 0x14 to a selected address. That address’s own offset-0x14 word is retained before the field is overwritten with twice the pre-traversal destination.

The retained pre-overwrite word governs subsequent routing. Its offset-0x14 observation, multiplied by the same signed halfword, determines a **product region**. That region’s offset-0x74 word is retained in invocation storage. Multiplying the product-region address by the halfword determines a **further region**; multiplying again selects the final offset-0x44 observation.

Three more self-address publications occur at the halfword-derived address’s offset 0x54, making five in total. Between the third and fourth, tenth-source offset 0x11 receives the retained pre-overwrite word’s low byte. Following the fourth, that field receives a byte with no established preceding assignment.

The late memory dependencies are:

- The further region’s first offset-0x44 observation precedes the first-input-offset-0x74 observation.
- Tenth-source offset 0x64 receives the retained product-region word before tenth-source offset 0x09 receives the first further-region observation’s low byte.
- A fresh further-region-offset-0x44 observation supplies that region’s offset-0x11 byte.
- The fifth halfword-address self-publication precedes a byte publication at twice the further-region address plus 0x11, using the retained product-region word’s low byte.
- The final product-selected offset-0x44 observation precedes the first-input-offset-0x14 observation.
- The retained first-input-offset-0x74 word’s low byte goes to offset 0x01 through the retained product-region word’s address.
- The newly observed first-input-offset-0x14 address receives the final selected word’s low byte at offset 0x11 and twice the further-region address’s low byte at offset 0x09.

The later assignment of a known byte does not define the earlier unassigned-byte consumption.

### Calls and terminal continuation

Before the incoming call, represented invocation storage has:

| Content | Number of word locations |
|---|---:|
| First input shifted left four bits | 2 |
| First input shifted left eight bits | 4 |
| Retained product-region-offset-0x74 word | 3 |
| Zero | 4 |

One retained-word location is populated during traversal. Final preparation assigns one location for each shifted value, then four zero locations, then the remaining copies. This does not establish a complete frame layout or additional explicit call arguments.

The incoming callable receives the fifth input’s storage address and two zeros.

If it returns, the seventh input’s storage is zeroed. Another invocation word receives 0x08400E1C, and three further words receive zero. The separate embedded source is observed afterward and passed to F-cdec09f5d432 with three zeros.

If that dependency returns, an exceptional primitive is reached with represented operands zero and 0x08401154. A callable transfer through its result is represented if such a result exists. Ordinary success is not established.

## Side effects

Any reached prefix of memory effects may remain if a later access fails or a dependency does not return. No rollback, explicit error result, or recovery path is represented.

No allocation, release, reference-count change, or ownership transfer is directly established. Dependency behavior remains unconstrained.

The incoming callable receives invocation-local storage. Its permitted access extent, retention rights, and any lifetime extension are unknown. Address exposure alone does not establish ownership transfer.

## Dependencies

The unavailable dependency identities use contract-local opaque IDs; these do not assert linkage identities.

| Opaque ID | Visible interface | Unresolved behavior |
|---|---|---|
| F-unknown-01 | Incoming callable; fifth-input storage address, zero, zero | Calling convention, memory reach, ownership, retention, return behavior |
| F-cdec09f5d432 | Post-call embedded-source observation, zero, zero, zero | Argument meanings, effects, ownership, return behavior |
| F-unknown-02 | Exceptional primitive; zero and 0x08401154 | Exceptional semantics and possible result-directed callable transfer |

No dependency is assumed to preserve globals, embedded data, argument storage, or reachable memory. In particular, the post-call embedded observation cannot be replaced by an entry snapshot.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes the relevant point is reached and preceding operations permit continuation.

1. **Intermediate visibility:** With fourth input 0x7E and first-input-offset-0x04 observation 0x123456AB, entry destination offset 0x11 receives 0x7E before 0xAB. Its offset 0x54 receives 0x123456AB before the destination address.

2. **Narrow publication:** A retained context-offset-0x14 value of 0x1234ABCD contributes halfword pattern 0xABCD to the early computed-address write, not a word publication.

3. **Signed routing:** Halfword pattern 0xFFFF contributes −1 to dependent arithmetic, subject to unresolved overflow and address validity.

4. **Byte interpretation:** A byte pattern 0xFF may widen to a value producing halfword pattern 0xFFFF or 0x00FF. The contract does not select between them.

5. **Freshness:** If an intervening overlapping write changes first-input offset 0x64, later consumers use their own observations. Previously retained consumers remain unchanged.

6. **Source observations:** If a source’s retained address observation differs from its publication observation under applicable memory semantics, the associated publication uses the latter. The ninth portion’s extra publication instead uses its retained address.

7. **Retained traversal state:** Overwriting the traversal-selected offset-0x14 field does not replace the saved word governing subsequent products.

8. **Late aliasing:** If tenth-source offset 0x09 overlaps the further region’s offset-0x44 word, its byte publication can affect the fresh word observation according to target byte order.

9. **Unassigned state:** The earlier unassigned-byte consumption is neither specified as zero nor supplied by its later assignment.

10. **Control flow:** A nonreturning incoming callable prevents post-call preparation and F-cdec09f5d432 invocation. If both calls return, ordinary successful return is not an equivalent terminal behavior.

## Uncertainties

The following remain **documented_only**:

- Application purpose and object relationships.
- Reliability of the represented function boundary and conventional interface.
- Origins and validity of implicit incoming state.
- Complete invocation-storage layout and dependency-visible extent.
- Embedded-source stability and alias relationships.
- Plain-byte signedness, byte order, alignment, and complete arithmetic semantics.
- Validity of negative, small, sum-derived, doubled, and product-derived addresses.
- Consequences of consuming the byte without a preceding assignment.
- Dependency memory effects, ownership, retention, and return behavior.
- Hardware observation and concurrency semantics.
- Exact exceptional-terminal behavior.

These gaps do not authorize invented validation, wrapping arithmetic, benign dependency assumptions, default byte values, or guaranteed completion.

## Evidence

**Established static evidence:** access widths and offsets; signed-halfword arithmetic; truncating publications; retained-versus-fresh observations; mutation ordering and address dependencies; invocation-storage assignments; visible call arguments; and the exceptional terminal continuation.

**Inferred intent:** propagation or linkage maintenance is plausible. No particular structure or application operation is established.

**Evidence limitations:** supplied warnings identify excluded unreachable regions and overlapping global symbols. They limit confidence in boundary and data interpretation without establishing additional reachable behavior.

No runtime tests have been performed. No execution trace, dependency implementation, complete memory map, or complete target calling convention was supplied.
