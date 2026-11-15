<!-- Trusted provenance: F-61e922cda7ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-61e922cda7ea

## Purpose

F-61e922cda7ea changes interconnected memory through eighteen ordered publication phases, a product-dependent traversal, and external interactions. Its application-level purpose is not established.

The central behavioral requirement is **preserving observations and their order when storage overlaps**. A write can change a later address, payload, or destination. Previously captured values remain effective unless the behavior explicitly requires another observation. An implementation equivalent only for disjoint storage would not satisfy this contract.

“Publication,” “selection,” “active address,” “anchor,” and “preserved word” name behavioral roles, not object types. They imply neither ownership nor synchronization.

This contract describes static evidence. It does not establish that any complete execution is valid or reaches its final interaction.

## Interface

### Explicit inputs

| Input position | Represented width | Established role |
|---|---:|---|
| First | Signed 32 bits | Numerical payload, memory-address contribution, byte-conversion source |
| Second | 32 bits | Incoming value has no represented direct use |
| Third | Signed 32 bits | Address contribution to the opening halfword write |
| Fourth | 8 bits | First active-offset-17 payload |
| Fifth | 32 bits | Address of its activation storage is passed externally |
| Sixth | 32 bits | Incoming value has no represented direct use |
| Seventh | 32 bits | Its activation storage is cleared after the first interaction returns |

Additional entry state supplies three address-bearing 32-bit integers, a 32-bit payload, and an invocation target. The integers initially provide the publication base, opening link, and active address. Their origins and calling convention are **documented_only**.

No return payload is represented. An ordinary return to the original caller is not established.

### Memory and arithmetic model

All offsets are byte offsets. Access widths are 8-bit bytes, 16-bit halfwords, and 32-bit words. Displacements and the traversal factor are signed 16-bit observations promoted for arithmetic. Character observations occupy eight bits, but their numerical signedness is unresolved.

Address-bearing calculations use represented 32-bit integers. Signed addition, multiplication, left shift, narrowing, and integer-to-address conversion require target rules; unrestricted modular arithmetic is not assumed. In particular, multiplication by two and left shift by one are distinct operations. Character interpretation can affect both output values and subsequent addresses.

Every reached access requires valid storage, permissions, bounds, and alignment under the target’s rules. No represented validation supplies those guarantees. There is no disjointness precondition between the named memory roles.

## Behavior

### Publication framework

There are eighteen publication phases, each associated with its own fixed word-source location. Phase numbers identify observable boundaries, not a required implementation structure.

Each phase captures its initial selection from its fixed source. A separate observation of that source is published at incoming publication base plus 68. The incoming retained word is then installed at captured selection plus 20. These are distinct source observations; equality cannot be presumed if the source changes between them.

Before phase 1, the opening publication base is converted to a halfword and written at third input plus opening link. Only afterward is the initial retained word observed at opening link plus 68. The opening active address is then written at its own offset 84.

For phases 1–17, the phase’s halfword output precedes a fresh retained-word observation at final selection plus 68. That observation precedes a word installation of the current active address at its own offset 84. Phase 18 has no corresponding halfword output or conclusion.

The next publication base is the preceding output’s numerical payload **before conversion to a halfword**. It is not reconstructed from the stored halfword.

| Phase | Halfword payload before conversion |
|---|---|
| 1–2 | Word captured at captured link plus 20 |
| 3 | Word captured at active address plus 20 |
| 4–5 | Captured link |
| 6–8 | Word captured at selection plus 20 |
| 9–10 | Captured link |
| 11–13 | Numerically interpreted character |
| 14–17 | Captured link |

Except in phase 8, the output address combines final selection with a signed displacement captured at active address plus original selection. Subsequent selection or active-address replacement does not refresh that displacement. Phase 8 instead combines selection with twice the opening active address.

Where used, the captured link is a word observed at active address plus 68. Discoveries and later active-address replacements do not retroactively change it.

### Input-derived effects and early payloads

The numerical first input, words observed at first input plus 100, and the additional entry payload are separate channels:

- Phase 2 installs the numerical first input at selection plus 20.
- First-input-offset-100 observations supply byte conversions, not substitutes for that numerical input.
- Phase 1 installs the entry payload at selection plus 36. A subsequent observation at selection plus 116 supplies phase 2’s offset-36 installation.
- Phase 2 captures its own selection-offset-116 word after that installation and before installing the first input at offset 20. This second capture later supplies phase 4’s word at first input plus 84 and phase 5’s byte at selection plus 13.

In phase 1, the fourth-input byte reaches active offset 17 before the first-input-offset-100 observation. That observation supplies both phase 1’s selection-offset-13 byte and phase 2’s initial active byte. Phase 1’s offset-116 capture precedes its selection-offset-13 write.

In phases 1 and 2, link and displacement capture precede the retained-word byte installation at first input plus 21. The linked-offset-20 payload is observed afterward. Consequently, overlap can change that payload without changing the captured link or displacement.

Before phase 2’s output, active offset 17 receives the first input’s byte conversion; the captured output-payload address plus 9 then receives the selection’s byte conversion.

Phase 3 writes the shifted-active-address byte at active offset 17 before capturing its displacement. Its retained-word byte at first input plus 9 precedes the active-offset-20 payload observation. A fresh first-input-offset-100 observation supplies another active-offset-17 byte before output.

Phase 4’s shifted-active byte precedes a fresh first-input-offset-100 observation, which precedes link and displacement capture. The observed word supplies phase 5’s initial active byte. Phase 4’s remaining pre-output writes have this order: the second offset-116 capture at first input plus 84; opening-active-address byte at selection plus 9; selection byte at captured link plus 13.

Phase 5’s additional byte destinations, in order, are selection plus 1, selection plus 13, and captured link plus 9. Their payloads derive respectively from selection, the second offset-116 capture, and selection.

### Discoveries and anchor establishment

Phases 3, 4, 6–8, 11–13, and 16 write a byte derived from active address shifted left by one at active offset 17.

Phases 6–8 discover addresses by observing a word at **signed displacement plus 68**. Selection is not an additional address contribution.

In phase 6, displacement capture follows the shifted-active byte. Discovery and selection-offset-20 payload capture both precede the fresh first-input-offset-100 observation used for another active byte.

Phase 7 observes offset 100 of phase 6’s discovery after its own shifted-active byte. Link and displacement capture precede the old active-address installation at incoming retained word plus 84. The discovery-derived byte reaches active offset 17 twice. Between those two writes, captured link plus 68 receives selection, another displacement-based address is discovered, and the selection-offset-20 payload is captured.

Phase 8’s discovery establishes the **anchor** before its selection-offset-20 payload capture. A word at offset 100 of phase 7’s discovery supplies a later active byte. Anchor offset 4 is captured before selection offset 116 establishes the **bridge word**. The offset-4 capture is installed at opening active address plus 84 before the exceptional halfword output. The concluding self-address installation remains a separate write and can overwrite it.

### Address replacement

| Phase | Selection replacement | Active-address replacement |
|---|---|---|
| 9 | None | Word at original selection plus 20 |
| 10 | Word at incoming retained word plus 20 | Word at original selection plus 20 |
| 14 | Word at incoming retained word plus 116 | Word at previous active address plus 20 |
| 15 | Word at captured link plus 116 | None |
| 16–17 | None | Word at signed displacement plus 20 |

Phase 9’s initial active byte comes from the anchor-offset-4 capture. A fresh anchor-offset-100 observation precedes link and displacement capture and supplies phase 10’s initial active byte.

In phases 9 and 10, bridge-word installation at anchor plus 84 falls between link/displacement capture and replacement-active-address observation. The original selection’s byte then reaches replacement active address plus 13.

Phase 10 captures its replacement selection after that byte write. Before output, bridge word plus 84 receives the anchor. The output still uses the displacement associated with the original selection and earlier active address.

Phase 14 begins with the byte carried from phase 13 and freshly observes anchor offset 100 before link and displacement capture. Selection replacement precedes active-address replacement. Captured link plus 13 receives the replacement selection’s byte.

In phase 15, the original selection’s byte reaches incoming retained word plus 21, followed by retained-word installation at original selection plus 68. Both precede replacement-selection observation at captured link plus 116. Either can affect that observation through overlap.

### Character-derived addresses and preserved state

Phases 11–13 capture a character at incoming retained word plus signed displacement. Its interpreted numerical value supplies the halfword payload and next publication base. In phases 12 and 13 it also supplies the base for an offset-13 byte write. Unresolved character signedness therefore affects addresses, not merely output representation.

Phase 11 captures the character before installing the bridge word’s byte at selection plus 29.

Phase 12 captures the character before freshly observing anchor offset 100. The observed word reaches signed displacement plus 84. The retained word’s byte then reaches character-derived base plus 13. Only afterward does selection offset 100 establish the **preserved word**.

Phase 13 observes anchor offset 100 before displacement and character capture. Its pre-output writes are ordered as follows:

| Destination | Payload |
|---|---|
| Anchor plus 1 | Selection byte |
| Selection plus 13 | Retained-word byte |
| Selection plus 68 | Preserved word |
| Character-derived base plus 13 | Selection byte |

The concluding selection-offset-68 observation remains fresh. It need not equal the preserved word after overlapping byte or halfword writes.

Fresh anchor-offset-100 observations carry active-byte payloads from phases 13 to 14, 14 to 15, 16 to 17, and 17 to 18. Phase 15 uses the byte carried from phase 14.

### Sum-addressed installations

In phases 16 and 17, the preserved word is installed at **selection plus incoming retained word**. Neither contribution is a fixed field offset.

Both phases capture link and displacement before replacing the active address. Captured link plus 84 then receives that link. The sum-addressed installation precedes the captured-link installation at selection plus 20.

A subsequent fresh word observation at preserved word plus 68 supplies the selection-offset-13 byte. Replacement active address plus 13 receives the captured link’s byte. The halfword output continues to use the previously captured link and displacement.

Phase 17 has three additional effects between capture and active-address replacement, in this order:

- Incoming retained word plus 84 receives the old active address.
- Old active address plus 17 receives a byte without established initialization.
- Captured link plus 68 receives selection.

A later assignment to the represented byte storage does not initialize its earlier use.

### Product-dependent traversal

Phase 18 publishes its fixed-source word, installs the incoming retained word at selection plus 20, and writes the carried anchor-offset-100 byte at active offset 17. Its active-offset-68 link and signed-halfword **factor** are captured afterward.

The factor serves both as multiplier and as an address base. There are five distinct word installations of the factor at factor plus 84.

The initial product of factor and captured link identifies an offset-68 word observation. Following that result through offset 116 and then offset 20 establishes a reached address. The reached address’s offset-20 word is captured before that location receives active address shifted left by one.

Traversal follows the **old capture**, not the replacement. A word at old capture plus 20, multiplied by the factor, establishes an intermediate address. Intermediate address plus 116 supplies the **saved traversal word**. Only afterward is the intermediate address multiplied by the factor again, establishing the **working product**.

The first two factor-base installations alternate with two selection-offset-17 writes carrying the active address’s byte conversion. After traversal reaches the intermediate address, the third factor installation and a selection-offset-17 byte carrying the old capture’s conversion precede the saved traversal observation. The fourth factor installation follows working-product establishment. The next selection-offset-17 byte has no established initialization.

Working product plus 68 and anchor plus 116 are captured before selection offset 100 receives the saved traversal word and selection offset 9 receives the earlier working-product capture’s byte. A **fresh** working-product-offset-68 observation then supplies working product’s offset-17 byte.

The fifth factor installation precedes a byte write at twice the working product plus 17 carrying the saved traversal word’s conversion. A later word observation at factor times working product plus 68 supplies another byte.

Finally, anchor offset 20 supplies a captured destination before saved traversal word plus 1 receives the earlier anchor-offset-116 capture’s byte. The captured destination receives the later product-dependent byte at offset 17, then the byte conversion of twice the working product at offset 9. Neither write refreshes that destination.

### Activation and continuation

Activation-associated assignments include two carrying anchor shifted left by four, four carrying anchor shifted left by eight, and three carrying the saved traversal word, counting its initial capture.

The initial shift-by-four capture precedes replacement of the anchor’s activation capture with its shift-by-eight value. Four word clears follow those initial shifted captures and precede the remaining copies. This does not establish a contiguous argument block or permission for a dependency to access it.

F-000000000001 is invoked through the entry-supplied target with fifth-input storage address and two zeros. After it returns, seventh-input storage is cleared. Another activation-associated word receives marker 0x08400E1C, followed by three further word clears.

A fixed-location word is then observed and passed to F-cdec09f5d432 with three zeros. After that interaction returns, F-000000000002 receives zero and marker 0x08401154. Its represented result supplies the target for F-000000000003, with no explicit arguments represented.

## Side effects

Writes affect input-derived, selected, linked, displacement-derived, character-derived, sum-addressed, doubled, product-derived, and activation-associated storage. All may overlap.

No allocation, release, ownership transfer, retention permission, synchronization, rollback, or recoverable-error result is established. Passing fifth-input storage address does not establish permission to release it or access adjacent storage.

Dependencies are not known to preserve globals, activation storage, or reachable memory. The fixed-location argument to F-cdec09f5d432 is observed after F-000000000001 returns, not before it.

Invalid accesses, unsupported arithmetic, uninitialized-value use, invalid invocation targets, and nonreturning interactions have no established recovery behavior. Earlier effects have no rollback guarantee. Repeated writes remain separate represented occurrences, without an established guarantee about atomicity, physical transactions, or concurrent visibility.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-000000000001 | Entry-supplied target; fifth-input storage address and two zeros | Calling convention, access extent, retention, ownership, preservation, effects, return |
| F-cdec09f5d432 | Later fixed-location word observation and three zeros | Argument meaning, ownership, preservation, effects, return |
| F-000000000002 | Undefined-instruction operation represented with zero and 0x08401154 | Trap semantics, result, continuation; ordinary-call behavior is not established |
| F-000000000003 | Invocation through the preceding result; no explicit arguments represented | Target validity, implicit inputs, effects, outcome |

Synthetic IDs identify interaction roles, not necessarily separate implementations.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes the relevant path is reached and target rules permit its accesses and arithmetic.

1. **Opening overlap:** If the opening halfword overlaps opening link plus 68, the initial retained-word observation incorporates that write according to target byte order.

2. **Independent source observations:** If a publication source changes between observations, captured selection and published word may differ. Replacing both observations with one cached value is not generally equivalent.

3. **Input identity:** If first input plus 100 contains a different numerical value from the first input, phase 2 nevertheless installs the numerical first input at selection plus 20.

4. **Mutable linked payload:** A first-input-plus-21 byte write overlapping the linked-offset-20 payload can change phase 1 or 2’s output without changing its already captured link or displacement.

5. **Signed contribution:** A displacement observed as −2 contributes −2, not 65534. Later selection replacement does not trigger displacement recapture.

6. **Exceptional output address:** Phase 8 uses twice the opening active address, not the displacement used to discover its anchor.

7. **Replacement-sensitive overlap:** If phase 15’s original-selection-offset-68 installation overlaps captured link plus 116, the subsequent replacement-selection observation reflects that installation.

8. **Character interpretation:** A high-bit-set character can produce different publication bases and offset-13 destinations under signed and unsigned interpretations. This contract does not select either interpretation.

9. **Fresh preserved source:** If a sum-addressed installation overlaps preserved word plus 68, the later byte source reflects the subsequent observation rather than a cached earlier word.

10. **Traversal stability:** Replacing the reached offset-20 word with the shifted active address does not redirect the next traversal step through that replacement; the old capture remains controlling.

11. **Multiplication boundary:** The saved traversal word is observed relative to the intermediate address before further multiplication. Later writes overlapping working product plus 68 can change its refreshed byte without changing its earlier capture.

12. **Uninitialized bytes:** Neither byte lacking established initialization may be assumed zero merely because nearby storage is later cleared or assigned.

13. **External ordering:** If F-000000000001 does not return, later clears and interactions are not reached. If it changes the later fixed source and returns, F-cdec09f5d432 receives the subsequent observation.

## Uncertainties

The following remain **documented_only**:

- Application intent and completeness of represented control flow.
- Origins and calling convention of additional entry state.
- Exact physical identities of fixed sources and activation storage beyond their represented roles.
- Both bytes lacking established initialization and their execution consequences.
- Storage validity, alignment, byte order, and integer-to-address conversion.
- Character signedness and target arithmetic, shift, and narrowing rules.
- Activation-storage accessibility, layout, and lifetime.
- Dependency ownership, retention, preservation, side effects, and return behavior.
- Concurrency, interruption, access granularity, and device-memory semantics.
- Undefined-instruction handling, final target validity, and ultimate return.

An uninitialized byte is not contractually zero, random, or a stable unspecified constant. Embedded warnings about omitted regions do not establish additional reachable behavior.

## Evidence

**Established static evidence:** The supplied representation supports the access widths, eighteen publication phases, separate fixed-source observations, signed-halfword contributions, capture-versus-refresh distinctions, replacement-sensitive addresses, repeated installations, traversal multiplication boundary, activation assignments, and external-interaction ordering described here.

**Inferred intent:** Publication-like effects and dependent traversal suggest manipulation of interconnected state. They do not establish a data-structure type, application operation, ownership discipline, or synchronization protocol.

**Not established:** Prior prose, reviewer feedback, and embedded comments are not independent behavioral evidence. Dependency implementations, target arithmetic specifications, instruction-level validation, and execution traces are unavailable.

No runtime tests have been performed.
