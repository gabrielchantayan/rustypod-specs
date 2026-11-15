<!-- Trusted provenance: F-b0e3cd0b9c34; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b0e3cd0b9c34

## Purpose

F-b0e3cd0b9c34 performs an ordered, mixed-width memory transformation, prepares argument-associated and stack-associated storage, invokes dependencies, and proceeds to a terminal control transfer.

No application-level purpose is established. Repeated offsets and address-valued payloads suggest relationships between memory regions, but do not establish a recognized object layout, ownership model, initialization protocol, or successful-return condition.

The contract concerns access history as well as final contents. Retained observations, fresh reads, overwritten writes, and changing destination bases are distinguishable when memory overlaps.

## Interface

### Represented arguments

| Position | Width | Visible use |
|---|---:|---|
| 1 | 32 bits | Numerical base for an early word read at byte offset 0x64. |
| 2 | 8 bits | Early indirect byte-store payload. |
| 3 | 32 bits | No direct value consumption is visible. |
| 4 | 8 bits | Payload of the first visible store. |
| 5 | 32 bits | Its associated storage is passed by address to an incoming executable target. |
| 6 | 32 bits | No direct value consumption is visible. |
| 7 | 32 bits | Its associated storage receives zero after that invocation returns. |

These widths do not establish physical argument placement, byte extension, or an external calling convention. Arguments 3 and 6 cannot be declared irrelevant: the dependency receives an address into argument-associated storage, and its access extent is unknown.

Additional incoming state comprises two 32-bit numerical address operands, a 32-bit payload, and an executable target. Below, the two address operands are the **original primary** and **original secondary** addresses. Three byte sources are consumed before any visible definition.

Execution also depends on directly addressed memory, indirectly selected memory, and fixed-location words.

### Results and preconditions

No ordinary return value or normal-return path is established. Observable results include memory reads and writes, exposed storage, dependency invocations, and terminal control behavior.

There is no visible validation, error code, rollback, or complete safe-input domain. Continuing execution requires every preceding access and dependency to permit progress. Numerical use as an address does not establish validity, alignment, allocation membership, lifetime, or disjointness.

## Behavior

### Reading conventions and retained roles

All offsets are byte offsets. A **word** is 32 bits; a **halfword** is 16 bits; a **byte** is 8 bits. Halfword observations used arithmetically are signed and extended before arithmetic. “Byte of” and “halfword of” describe the represented narrowing conversion, not an independently established target conversion rule.

The **active region** initially has the original primary address. It changes only where stated below. A **selector** is the retained address obtained at a fixed-location transition. A **relationship region** is another retained numerical address whose source changes during the transformation. Neither term implies an object type.

Descriptions preserve ordering between successive operations. They do not establish operand-evaluation order within a single expression, atomicity, synchronization, or one physical transaction per represented access.

### Fixed-location handoffs

There are twenty-one successive fixed-location handoffs. Numbering identifies the fixed-word sites by their order in this body; it does not imply contiguous storage or nonoverlap.

Each handoff has this ordered behavior:

1. The specified region receives its own address as a word at offset 0x54.
2. The handoff’s fixed word is read and retained as the **next selector**.
3. That same fixed word is read again, freshly. This second observation is written as a word at offset 0x44 of the specified destination.
4. The previously captured handoff payload is written at offset 0x14 of the retained next selector.
5. The stated post-handoff byte store occurs.

The first fixed-word observation determines the offset-0x14 destination; the second supplies the offset-0x44 payload. They must not be replaced by one cached observation. Neither observation is replaced by a subsequent reread of the destination field. The offset-0x44 store may overlap the fixed cell or the retained selector’s storage without changing the already captured selector address.

The following progression specifies the payloads, destinations, intervening accesses, and active-region changes.

### Initial selection and handoffs 1–4

**1. Initial argument effects.** Argument 4 is first stored at original-primary offset 0x11. The word at argument-1-relative offset 0x64 is then captured. The original primary’s offset-0x44 word selects an initial destination, and a signed halfword is captured at the sum of the two original addresses.

The selected destination receives its own address at offset 0x34 and argument 2 at offset 0x0D. The original primary receives a byte of that destination address at offset 0x0D. A halfword of the destination address is stored at the captured signed halfword plus the original secondary address. Only then is the original secondary’s offset-0x44 word captured.

Handoff 1 uses that last word as payload, the initial destination as its offset-0x44 destination, and the original primary for its self-address store. Afterwards, original-primary offset 0x11 receives a byte of the retained argument-1-relative word.

**2. First relationship capture.** The original primary’s offset-0x44 word is reread to select a destination. A halfword at original primary plus selector is captured. The destination receives its own address at offset 0x34 and a byte of the earlier original-secondary observation at offset 0x0D. The original primary’s offset-0x14 word becomes the relationship-region address.

The destination receives its own address at offset 0x54. A halfword of its address is stored at the captured halfword plus selector. The selector’s offset-0x44 word becomes the handoff payload. Handoff 2 targets that destination and uses the original primary’s self-address store. Afterwards, original-primary offset 0x11 receives a byte of twice the original primary address.

**3. Tripled-halfword destination.** The relationship region’s offset-0x64 word is retained. A destination is selected freshly from original-primary offset 0x44, followed by a halfword capture at original primary plus selector.

A byte of the preceding handoff payload is stored at selector plus that payload. The selected destination’s offset 0x0D receives a byte from a fresh original-primary offset-0x64 word read. A halfword of the destination address is stored at three times the captured signed halfword. A word at twice that halfword plus 0x44 becomes the new relationship-region address.

Handoff 3 propagates that address to the next selector’s offset 0x14, targeting the selected destination and using the original primary’s self-address store. Its post-handoff byte is from the retained offset-0x64 observation.

**4. Address source replaced by a word observation.** After a fresh original-primary offset-0x44 selection and halfword capture at original primary plus selector, a byte of the relationship address is stored at selector plus that relationship address.

A fresh original-primary offset-0x64 word is then used both as a byte payload at the selected destination’s offset 0x0D and as a numerical address. The relationship address is replaced by a fresh original-primary offset-0x14 observation. The destination receives its own address at offset 0x54. Its narrowed address is stored as a halfword at the captured halfword plus the offset-0x64 numerical value.

A word at that numerical value plus 0x44 supplies handoff 4’s payload. The handoff targets the selected destination, uses the original primary’s self-address store, and is followed by a byte of twice the original primary at offset 0x11.

### Payload relay and handoffs 5–9

**5. Incoming payload exposure.** A relationship-relative offset-0x64 word is retained. After fresh primary-relative destination selection and halfword capture, the relationship region receives a byte of the previous handoff payload at offset 0x15. The selected destination is redirected through its offset-0x14 word.

The current selector receives the incoming 32-bit payload at offset 0x24. Its offset-0x74 word is retained as the **relay payload**, then its offset 0x0D receives a byte of the retained relationship-relative offset-0x64 observation. A halfword of the redirected destination is stored at captured halfword plus selector.

The selector’s fresh offset-0x44 word supplies handoff 5. The handoff targets the redirected destination, uses the original primary’s self-address store, and is followed by the same retained offset-0x64-derived byte at primary offset 0x11.

**6. Relay replacement.** After fresh primary-relative selection and halfword capture, relationship offset 0x15 receives a byte of the preceding handoff payload. Selection again follows the destination’s offset-0x14 word.

The selector receives the old relay payload at offset 0x24, after which its offset-0x74 word replaces the retained relay payload. Selector offset 0x14 receives the relationship address; original-primary offset 0x11 also receives a byte of that address. The redirected destination receives a byte of the selector at offset 0x09. The halfword store at captured halfword plus selector follows.

A fresh selector-relative offset-0x44 word supplies handoff 6. Its destination is the redirected destination; its self-address store is primary-relative; its following byte is derived from twice the primary address.

**7. Fresh relationship read.** Following a halfword capture at primary plus selector, relationship offset 0x09 receives a byte of the preceding handoff payload. A fresh primary offset-0x14 observation selects the destination. Primary offset 0x11 receives a byte from a fresh relationship offset-0x64 read before the halfword store at captured halfword plus selector.

A fresh selector offset-0x44 observation supplies handoff 7. The handoff uses that destination and the primary self-address store, followed by a byte of twice the primary address.

**8. Relay written elsewhere.** A relationship offset-0x64 word is retained before fresh primary-relative selection and halfword capture. Relationship offset 0x54 receives the retained relay payload. Selector offset 0x09 receives a byte of the primary address, and destination offset 0x0D receives a byte of the selector. After the halfword store, a fresh selector offset-0x44 observation supplies handoff 8. The post-handoff primary byte comes from the retained offset-0x64 word.

**9. Selector-derived bytes.** Fresh primary-relative selection and halfword capture precede three byte stores: selector offset 0x01 receives a byte of itself; selector offset 0x0D receives a byte of the relay payload; destination offset 0x09 receives a byte of the selector. The halfword store follows.

A fresh selector offset-0x44 word supplies handoff 9. As in handoff 8, the self-address store is primary-relative. Its following primary byte is derived from twice the primary address.

### Anchor acquisition and handoffs 10–14

**10. Side-region selection.** A halfword at primary plus selector selects a **side region** through a word read at signed halfword plus 0x44. The selector’s offset-0x14 word selects the handoff destination. Primary offset 0x11 receives a byte from a fresh relationship offset-0x64 observation before the halfword store at captured halfword plus selector.

The selector’s offset-0x44 word becomes the new relationship address and handoff payload. Handoff 10 uses the selected destination and primary self-address store, followed by a byte of twice the primary address.

**11. First unresolved byte.** A side-region offset-0x64 word is retained before fresh primary-relative selection and halfword capture. Relationship offset 0x54 receives the primary address. Primary offset 0x11 then receives the first byte source lacking a visible prior definition.

The selected destination’s offset 0x44 receives the retained selector address directly; this is not another fixed-location read. A word at captured halfword plus 0x44 replaces the relationship address. A fresh selector offset-0x14 word redirects the destination. Primary offset 0x11 receives a byte of the retained side-region offset-0x64 word, followed by the halfword store.

A fresh selector offset-0x44 observation supplies handoff 11. It uses the redirected destination and primary self-address store, followed by a byte of twice the primary address.

**12. Anchor and bridge.** A halfword at primary plus selector selects the persistent **anchor region** through a word read at signed halfword plus 0x44. Selector offset 0x14 selects the destination. Primary offset 0x11 receives a byte from a fresh relationship offset-0x64 observation.

Anchor offset 0x04 is captured. Selector offset 0x74 supplies a retained **bridge address**. Primary offset 0x54 receives the captured anchor word. A halfword of the destination is then stored at twice the primary address plus selector. The selector’s offset-0x44 word is captured before primary offset 0x54 is restored to the primary address.

Handoff 12 propagates that captured selector word, targets the selected destination, and is followed by a primary-offset-0x11 byte from the anchor offset-0x04 observation.

**13. First active-region change.** Anchor offset 0x64 is retained before fresh primary-relative selection and halfword capture. Anchor offset 0x54 receives the bridge address. Selector offset 0x14 selects the new active region, whose offset 0x0D receives a byte of the selector. The halfword store still uses the previously captured halfword plus selector and the earlier selected destination.

Selector offset 0x44 becomes the relationship address. Handoff 13 targets the earlier destination but uses the new active region for its self-address store. New-active offset 0x11 then receives a byte of the retained anchor offset-0x64 observation.

**14. Cross-region relationship.** Selection and halfword capture now use the active region. Anchor offset 0x54 again receives the bridge address. A fresh selector offset-0x14 observation replaces the active region, whose offset 0x0D receives a byte of the selector.

The relationship region’s offset-0x14 word supplies a separate numerical base. Bridge offset 0x54 receives the anchor address. A halfword of the selected destination is stored at the captured halfword plus that separate base. The word at that base plus 0x44 becomes the new relationship address.

Handoff 14 propagates that address, targets the selected destination, and uses the new active region’s self-address store. Active offset 0x11 receives a byte of twice the active address.

### Byte-selected regions and handoffs 15–17

**15. Byte-addressed handoff.** After an active-plus-selector halfword capture, a character is read at captured halfword plus relationship address. Selector offset 0x1D receives a byte of the bridge address. A halfword of the promoted character is stored at captured halfword plus selector.

A fresh selector offset-0x44 word becomes the relationship address. Handoff 15 uses the promoted character as its destination base, the active region for its self-address store, and that relationship word as payload. It is followed by a byte of twice the active address at active offset 0x11.

**16. Retained offset-0x64 payload.** A signed halfword at active plus selector becomes a numerical base. A character is read at that base plus relationship address. A fresh anchor offset-0x64 word is stored at signed-halfword-relative offset 0x54. Character-relative offset 0x0D receives a byte of the relationship address.

The selector’s offset-0x64 word is then retained as the replacement bridge value. After a halfword store at signed halfword plus selector, a fresh selector offset-0x44 word becomes the relationship address. Handoff 16 targets the promoted character, uses the active self-address store, and is followed by a byte of twice the active address.

**17. Retained payload versus fresh propagation.** Anchor offset 0x64 is captured, followed by an active-plus-selector halfword and a character at that halfword plus relationship address. Anchor offset 0x01 receives a byte of the selector; selector offset 0x0D receives a byte of the relationship address.

Selector offset 0x44 receives the retained bridge value captured from the preceding selector’s offset 0x64. Intervening stores then write a byte of the selector at character-relative offset 0x0D and a halfword of the promoted character at captured halfword plus selector.

Only afterwards is selector offset 0x44 read freshly. This observation—not necessarily the earlier retained bridge value—becomes the relationship address and handoff 17 payload. The handoff targets the promoted character and uses the active self-address store. Its following active-offset-0x11 byte comes from the earlier anchor observation.

### Redirected active regions and handoffs 18–21

**18. Offset-0x74 redirection.** A fresh anchor offset-0x64 word is retained before active-relative selection and halfword capture. Relationship offset 0x74 supplies a separate base. Active offset 0x14 selects the replacement active region. The earlier selected destination receives a byte of the separate base at offset 0x0D, followed by a halfword store at captured halfword plus that base.

The word at separate base plus 0x44 becomes the relationship address and handoff 18 payload. The handoff targets the earlier destination, uses the replacement active region’s self-address store, and is followed by the retained anchor-derived byte.

**19. Another offset-0x74 route.** Fresh active-relative selection and halfword capture precede a byte of the selector at relationship offset 0x15. Selector offset 0x44 receives the relationship address. The selected destination’s offset-0x74 word then supplies a separate base.

After the halfword store at captured halfword plus that base, the base’s offset-0x44 word becomes the new relationship address. Handoff 19 targets the selected destination and uses the active self-address store. Its following byte is derived from twice the active address.

**20. Offset-0x14 active selection.** A fresh anchor offset-0x64 word is retained before active-relative selection and halfword capture. A word at signed halfword plus 0x14 selects the replacement active region.

The selected destination receives its own address at offset 0x54. A word store at selector plus relationship address writes the retained bridge value. Selector offset 0x14 receives the selected destination address. Selector offset 0x0D receives a byte from a fresh bridge-relative offset-0x44 read. Replacement-active offset 0x0D receives a byte of the selected destination, followed by the halfword store at captured halfword plus selector.

A fresh selector offset-0x44 word supplies handoff 20. The destination remains the earlier selected destination; the self-address store and following retained-anchor byte use the replacement active region.

**21. Second unresolved byte and repeated propagation.** A fresh anchor offset-0x64 word is retained before active-relative selection and halfword capture. Relationship offset 0x54 receives the current active address. Current-active offset 0x11 receives the second byte source lacking a visible prior definition. The selected destination’s offset 0x44 receives the retained selector address directly.

A word at captured halfword plus 0x14 then replaces the active region. The destination self-address store, bridge-value store at selector plus relationship, selector offset-0x14 replacement, fresh bridge-relative byte source, active offset-0x0D store, and halfword store occur in the same order described for handoff 20.

A fresh selector offset-0x44 observation supplies handoff 21. Its self-address store uses the replacement active region. The following active-offset-0x11 byte comes from the retained anchor observation.

### Product-dependent completion

The active region’s offset-0x44 word is captured. A signed halfword at active plus selector becomes a retained **factor** and numerical address base.

Factor-relative offset 0x54 receives the factor itself, then selector offset 0x11 receives a byte of the active address. This alternating pair occurs twice.

The factor multiplied by the captured active-relative word supplies a base for an offset-0x44 read. That observation selects an offset-0x74 read, which selects an offset-0x14 read. The resulting region’s own offset-0x14 word is captured before replacement with twice the active address. The captured pre-overwrite value selects another offset-0x14 word; multiplication by the factor establishes a product-derived base.

After another factor self-address store and a selector-offset-0x11 byte of the captured pre-overwrite value, the product-derived base’s offset-0x74 word is saved. Multiplying that base by the factor establishes the later product address.

Another factor self-address store precedes the third byte source lacking a visible prior definition, written at selector offset 0x11. The later product’s offset-0x44 word is captured, followed by anchor offset 0x74. Selector offset 0x64 receives the saved word, and selector offset 0x09 receives a byte of the captured product-relative word.

A fresh later-product offset-0x44 read supplies a byte at later-product offset 0x11. After another factor self-address store, twice the later product plus 0x11 receives a byte of the saved word.

A word at factor times later product plus 0x44 is captured. A fresh anchor offset-0x14 word selects the final destination. Saved-word-relative offset 0x01 receives a byte of the retained anchor offset-0x74 observation. The final destination then receives a byte of the further product-based observation at offset 0x11 and a byte of twice the later product at offset 0x09.

### Storage exposure and control transfer

The retained anchor address supplies results shifted left by four and eight bits. Those initial assignments precede four zero assignments. Subsequent copies produce, counting originals and copies, two storage assignments carrying the four-bit-shifted result, four carrying the eight-bit-shifted result, and two carrying the saved product-derived word.

The incoming executable target receives the address of argument-5-associated storage and two zeros.

If it returns, argument-7-associated storage receives zero. Another stack-associated location receives 0x08400E1C, followed by three zero assignments. A fixed-location word is then read freshly and passed to F-cdec09f5d432 with three zeros.

If that dependency returns, a terminal mechanism receives represented operands zero and 0x08401154. Its produced executable target is invoked. This sequence is annotated as nonreturning; no particular signal, exception, exit status, or recovery behavior is established.

## Side effects

Narrow stores do not independently clear adjacent bytes. Overlap can nevertheless change subsequent wider observations, selection words, or fixed-location reads.

Signed halfwords range from −32768 through 32767. Doubling and tripling that range fit signed 32-bit arithmetic. Adding arbitrary 32-bit operands, multiplying general address values, doubling addresses, and shifting left by one, four, or eight bits need not fit. No checks are visible, and universal modulo-\(2^{32}\) behavior is not established.

Character promotion can affect byte-selected addresses and halfword payloads. Endianness, alignment, integer-to-address conversion, narrowing, overflow, and negative or overflowing shifts remain **documented_only**.

No allocation, release, ownership transfer, or lifetime extension is established. Neither dependency may be assumed to preserve globals or allocations. Passing a storage address does not establish retention rights or permission to access neighboring locations.

Earlier completed effects may remain after a later failure or nonreturning invocation. Partial completion of an individual failing access is **documented_only**.

## Dependencies

- **Incoming executable dependency; opaque function ID unavailable.** Receives the address of argument-5-associated storage and two zeros. Identity, access extent, mutations, retention, ownership effects, exceptions, and return behavior are unresolved.
- **F-cdec09f5d432.** Receives a freshly read post-invocation fixed word and three zeros. No result is visibly consumed. Its semantic contract and preservation behavior are unresolved.
- **Terminal mechanism and produced executable target; opaque function IDs unavailable.** Their operand and invocation relationship is established above. Environment-level behavior is unresolved.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Early failure:** If the first store succeeds but the argument-1-relative word read fails, argument 4 has already been stored at original-primary offset 0x11.

2. **Separate fixed observations:** If a handoff’s two fixed-word reads yield different numerical values, the first determines the next selector and the second supplies the offset-0x44 payload. A single cached read is not equivalent.

3. **Retained destination:** If the handoff’s offset-0x44 store changes the fixed cell, the subsequent offset-0x14 store still uses the previously retained selector address.

4. **Middle propagation under overlap:** At handoff 17, if an intervening byte or halfword store changes the selector’s offset-0x44 contents, the propagated payload is the later fresh word observation, not automatically the retained offset-0x64-derived bridge value.

5. **Signed halfword:** A captured halfword of −1 yields −3 when tripled and 0x42 when doubled and increased by 0x44. These numerical results do not establish accessible addresses.

6. **Character interpretation:** A loaded pattern of 0xFF may promote to −1 or 255 under the unresolved character model, affecting byte-selected destinations.

7. **Pre-overwrite capture:** Replacing a product-selected offset-0x14 field does not replace its captured former value used to select the next multiplier.

8. **Late reread:** If intervening writes alter later-product offset 0x44, its second observation supplies the product-relative offset-0x11 byte.

9. **Dependency mutation:** If the incoming executable dependency changes the final fixed-word source and returns, F-cdec09f5d432 receives the later observation, subject to intervening effects.

10. **Incomplete inputs:** The seven represented arguments alone do not determine the three byte uses preceding visible definitions.

11. **Terminal progression:** If all preceding accesses complete and both dependencies return, execution reaches the terminal mechanism rather than an established successful return.

## Uncertainties

The following remain **documented_only**:

- Actual function boundary, calling convention, and additional incoming-state provenance.
- Argument-associated and stack-associated physical layout, aliasing, and caller visibility.
- Concrete validity, mutability, overlap, and allocation extent of addressed storage.
- Provenance and execution-level consequences of the three unresolved byte sources.
- Target arithmetic, conversion, character, alignment, and endianness rules.
- Dependency preservation, retention, allocation, release, exceptions, and nonlocal transfers.
- Terminal handling and behavior outside the supplied representation.

Unknown fixed-word contents do not make their visible read order unknown. Likewise, unknown aliasing does not justify merging retained and fresh observations.

## Evidence

**Established static evidence:** The supplied representation establishes the argument widths, additional incoming operands, mixed-width accesses, twenty-one two-observation fixed-location handoffs, changing active bases, retained payloads, intervening mutations, product-dependent sequence, storage preparation, dependency order, and terminal progression described above.

**Inferred intent:** Address-valued stores and recurring offsets may encode relationships between memory regions. No application-level object model or ownership discipline is established.

**Evidence limitations:** Annotations describe overlapping fixed-location symbols and omitted regions characterized as unreachable. They are not independent proof of layout or reachability. Dependency implementations and independent target-semantics evidence are unavailable.

No runtime tests have been performed.
