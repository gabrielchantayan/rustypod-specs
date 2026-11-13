<!-- Trusted provenance: F-4bbeb6c9fa4c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4bbeb6c9fa4c

## Purpose

F-4bbeb6c9fa4c performs an ordered, alias-sensitive memory transformation, replaces an indirectly discovered link, prepares frame storage, and enters a sequence of external interactions.

The established contract concerns memory accesses and control-transfer conditions. It does not establish a valid object model, application-level success condition, ownership transfer, or normal return.

Terms such as **discovery**, **replacement base**, **anchor**, and **payload** describe roles within this contract. They are not evidence that the addressed storage represents particular application objects.

## Interface

### Inputs and inherited state

| Input | Represented width | Established role |
|---|---:|---|
| Argument 1 | Signed 32 bits | Snapshot and payload addressing, final destination lookup, frame-preparation shifts |
| Argument 2 | Signed 32 bits | Opening character addressing |
| Argument 3 | Opaque 32 bits | No explicit use |
| Argument 4 | Opaque 8 bits | First memory mutation |
| Argument 5 | Opaque 32-bit argument storage | Its address is passed to an indirect callable |
| Argument 6 | Opaque 32 bits | No explicit use |
| Argument 7 | Opaque 32-bit argument storage | Cleared after the first external interaction returns |

Inherited state supplies an **initial base**, an **auxiliary base**, an opaque byte, and an indirect callable target. Both bases are represented as signed 32-bit values. Neither is established as Argument 1.

Seven external signed-word sources participate in the memory transformation. A separate external datum supplies an operand at a later call boundary; its width is unresolved.

The function has no explicit result value. Clearing Argument-7 storage does not by itself establish a caller-visible output.

### Widths and arithmetic

All offsets below are byte offsets. Character and opaque-byte accesses occupy 8 bits; signed-halfword accesses occupy 16 bits; signed-word and opaque-word accesses occupy 32 bits. The identified frame assignments are also 32 bits.

Signed halfwords contribute their sign-extended values to arithmetic. Plain-character signedness is unresolved, including when a captured character contributes to an address. Character conversion, signed-halfword conversion, and opaque-byte narrowing are distinct operations.

The represented signed arithmetic is 32-bit. Supported execution requires every reached addition, multiplication, and shift to have a supported interpretation; modular wrapping is not established. For nonnegative signed operands, left shifts by one, four, and eight require upper bounds of 0x3FFFFFFF, 0x07FFFFFF, and 0x007FFFFF respectively. Negative signed left shifts have no established supported interpretation. Multiplication by two remains multiplication, not an interchangeable shift.

Every reached address must support the indicated access. Alignment, byte order, address representation, unsupported conversions, and fault delivery remain **documented_only**.

## Behavior

### Observation model

The contract describes represented access ordering, not physical bus transactions, volatile semantics, atomicity, or synchronization.

Earlier mutations may affect later observations, including observations that determine addresses. Arguments, external sources, indirect destinations, and frame storage are not established as disjoint.

A captured value has no explicit refresh unless one is identified below. This is not a guarantee that its represented storage is immune to aliasing writes.

### Opening transformation

The opening establishes the first incoming word and first carrier character:

- Argument 4 reaches initial base +0x11 before the signed halfword at initial base plus auxiliary base is observed.
- That halfword contributes to the character address together with Argument 2.
- After character capture, the inherited byte reaches auxiliary base +0x1D. The character’s signed-halfword conversion then reaches captured halfword plus auxiliary base.
- Only afterward is the signed word at auxiliary base +0x44 observed as the first incoming word.

Thus the initial byte mutation can affect the opening halfword, and either subsequent mutation can affect the first incoming word.

### Seven-round publication protocol

The transformation contains seven rounds. Each has an active base, a carrier, an incoming value, and its own external signed-word source.

At round entry, the active base is first published as a signed word at active base +0x54. The external source is then referenced to obtain the **selector**. A separate reference to the same source supplies a signed-word publication at carrier +0x44. Incoming publication at selector +0x14 follows; the entry-byte mutation at active base +0x11 completes the entry protocol.

These references must not be collapsed into an assumed entry-time constant. They establish separate value dependencies, without establishing a physical read count.

| Round | Active base | Carrier | Entry-byte input |
|---|---|---|---|
| Discovery 1 | Initial base | Opening character | Active base shifted left once |
| Discovery 2 | Initial base | Discovery-1 character | Active base shifted left once |
| Redirection 1 | Initial base | Discovery-2 character | Discovery-2 snapshot |
| Redirection 2 | First replacement base | Redirection-1 companion | Redirection-1 snapshot |
| Reuse 1 | First replacement base | Redirection-2 companion | Active base shifted left once |
| Reuse 2 | Second replacement base | Reuse-1 companion | Reuse-1 snapshot |
| Terminal | Third replacement base | Reuse-2 companion | Retained Reuse-2 byte |

The first six entry-byte writes use character conversion. Terminal entry uses an opaque-byte write. Incoming values and publications are signed words through Reuse 2’s entry. Reuse 2’s outgoing observation and Terminal’s incoming publication are opaque words.

A **snapshot** is an opaque-word observation at Argument 1 +0x64. A **companion** is a signed-word observation at the round’s active base +0x44. Each nonterminal round supplies the next incoming value through its outgoing observation.

### Discovery and retained anchor

Both discovery rounds obtain a signed halfword from active base plus selector, then a character from captured halfword plus incoming value. Their closing mutation stores the character’s signed-halfword conversion at captured halfword plus selector. Their outgoing word is observed at selector +0x44 after closure.

Discovery 1 also establishes a retained **anchor address**. Character capture precedes an opaque-word copy from Argument 1 +0x64 to captured halfword +0x54. That copy precedes a character-converted incoming-value write at discovered character +0x0D. The anchor is then captured as a signed word from selector +0x64, before closure.

Consequently, the snapshot copy and character mutation can affect anchor capture through overlap. The anchor is a retained address value, not a promise about later contents at that address.

Discovery 2 captures its snapshot before its halfword and character. Between character capture and closure, its mutations have this precedence:

1. Selector’s character conversion reaches Argument 1 +1.
2. Incoming value’s character conversion reaches selector +0x0D.
3. The retained anchor reaches selector +0x44 as a signed word.
4. Selector’s character conversion reaches discovered character +0x0D.

Closure follows all four. The outgoing observation is therefore not necessarily the anchor just published.

### Redirection and first replacement base

Each redirection round captures its companion before its signed halfword from active base plus selector. Its closure stores the companion’s signed-halfword conversion at captured halfword plus a subsequently discovered **redirection address**. The outgoing signed word is observed at redirection address +0x44 after closure.

Redirection 1 captures a fresh snapshot before its companion. After halfword capture, it obtains the redirection address from incoming value +0x74, then the first replacement base from active base +0x14. Before closure, the redirection address’s character conversion reaches companion +0x0D.

Redirection 2 uses that replacement base and the snapshot retained by Redirection 1; it captures neither a new snapshot nor another replacement base. After companion and halfword capture, selector’s character conversion reaches incoming value +0x15. Incoming value is then published as a signed word at selector +0x44. Only after these mutations is the redirection address observed at companion +0x74.

Those intervening mutations may change the redirection lookup. They do not request fresh companion or halfword observations.

### Reuse and remaining replacement bases

Both reuse rounds capture a fresh snapshot, then their companion, then a signed halfword from active base plus selector. The signed word at captured halfword +0x14 supplies the next replacement base.

Reuse 1 selects its replacement base immediately after halfword capture.

Reuse 2 delays that selection until three mutations have completed, in order: active base is published at incoming value +0x54; the fresh snapshot is narrowed to an opaque byte and written at active base +0x11; selector is published at companion +0x44. The narrowed byte is retained for Terminal entry.

After replacement selection, both rounds impose the same publication and closure order:

- Companion reaches companion +0x54.
- Retained anchor reaches selector plus incoming value.
- Companion reaches selector +0x14.
- A fresh opaque word from retained anchor +0x44 supplies a character-converted write at selector +0x0D.
- Companion’s character conversion reaches replacement base +0x0D.
- Companion’s signed-halfword conversion reaches captured halfword plus selector.

The first three publications are signed words. The outgoing observation at selector +0x44 follows closure: signed in Reuse 1, opaque in Reuse 2.

The two anchor-content observations are separate. Neither recaptures the anchor address from its original field.

### Terminal link replacement and products

Following Terminal entry, companion capture precedes the signed-halfword observation at active base plus selector. This halfword becomes the **factor**, with range −32768 through 32767.

Five publication pairs delimit the remaining transformation. Each pair first publishes the factor as a signed word at numeric address factor +0x54, then performs the indicated character-converted write.

| Pair | Character input | Destination |
|---|---|---|
| 1 | Terminal active base | Terminal selector +0x11 |
| 2 | Terminal active base | Terminal selector +0x11 |
| 3 | Old link | Terminal selector +0x11 |
| 4 | Late-word storage value | Terminal selector +0x11 |
| 5 | Late-word storage value | Twice the product-derived base +0x11 |

Pairs 1 and 2 are distinct, ordered effects despite their matching operands and destinations.

After pair 2, three dependent signed-word observations locate a **link container**: the first uses factor multiplied by Terminal companion plus 0x44; the second uses that result plus 0x74; the third uses the second result plus 0x14.

The signed word at container +0x14 becomes the **old link**. That field is replaced with Terminal active base shifted left once. Only after replacement is the signed word at old link +0x14 observed. Its multiplication by the factor produces the **intermediate product**, before pair 3.

After pair 3, the signed word at intermediate product +0x74 is assigned to frame-relative +0x108, called **late-word storage**. Multiplication of the factor by the intermediate product then produces the **product-derived base**, before pair 4.

The late word is not an operand of that multiplication. Its storage nevertheless remains mutable frame storage, not an immutable mathematical value.

### Payload distribution

After pair 4, an opaque word at product-derived base +0x44 is captured before the opaque word at Argument 1 +0x74, the **primary payload**. Both captures precede signed-word publication of the late-word storage value at Terminal selector +0x64.

The first captured word supplies a character-converted write at Terminal selector +9. A later, separate opaque-word observation at product-derived base +0x44 supplies a character-converted write at product-derived base +0x11. Intervening writes may distinguish the two observations.

Pair 5 follows. An opaque word at factor multiplied by product-derived base plus 0x44 becomes the **final payload**. The signed word at Argument 1 +0x14 is then captured as the **destination**.

Three final character-converted writes follow in order:

| Destination address | Input |
|---|---|
| Late-word storage value +1 | Primary payload |
| Destination +0x11 | Final payload |
| Destination +9 | Twice the product-derived base |

Payloads and destination are not explicitly refreshed between these writes. Pair 5’s multiplication by two and the final write’s multiplication by two are separate operations.

### Frame preparation and continuation gates

Before external interaction, Argument 1 shifted left four positions is assigned at frame-relative +0x10, followed by its eight-position shift at +0x34.

Opaque-word frame locations +0xE4, +0x60, +0xA8, and +0x1B8 are cleared in that order. The following assignments then occur in table order:

| Frame destination | Frame source |
|---|---|
| +0x24 | +0x10 |
| +0x4C | +0x34 |
| +0x128 | Late-word storage |
| +0x13C | +0x34 |
| +0x17C | Late-word storage |
| +0x188 | +0x34 |

These offsets identify represented frame storage, not a verified portable structure or additional established call arguments.

F-8c2e7a109b35 receives the address of Argument-5 storage and two zero operands. No result is explicitly consumed. Its return gates all subsequent effects: Argument-7 storage is cleared only afterward.

Next, frame-relative +0x19C receives 0x08400E1C, followed by opaque-word clearing of +0x100, +0x120, and +0x140 in that order.

F-cdec09f5d432 receives the external datum and three zero operands. The datum is referenced at this boundary, not established as an entry-time snapshot.

If that interaction returns, F-d61f29c0a8e7 is reached with operands zero and 0x08401154. The representation uses its result as a callable target and attempts an invocation with no explicit arguments. Valid continuation, termination, and eventual return remain unresolved.

## Side effects

Established explicit effects include indirect byte, halfword, and word mutations; replacement of a discovered link field; frame assignments; Argument-7 clearing; exposure of Argument-5 storage; and external control transfers.

There is no explicit allocation, release, ownership transfer, or lifetime extension. Passing Argument-5 storage does not establish permission to retain its address. Conversely, absent dependency implementations, retention or other memory effects cannot be excluded.

No checked rejection, rollback, recovery path, or success indication is established. Unsupported arithmetic, inaccessible storage, or a nonreturning interaction may prevent later effects. Fault delivery and visibility of preceding mutations remain **documented_only**.

## Dependencies

| Opaque function ID | Established boundary |
|---|---|
| F-8c2e7a109b35 | Inherited indirect callable; receives an argument-storage address and two zeros |
| F-cdec09f5d432 | External callable; receives the boundary-referenced datum and three zeros |
| F-d61f29c0a8e7 | Undefined-instruction interaction with operands zero and 0x08401154; its represented result supplies a subsequent indirect target |

The first and third IDs label interaction roles, not independently resolved implementation identities. The subsequent callable has no established stable identity.

Calling conventions, unresolved operand widths, preservation of globals or frame storage, address retention, ownership effects, error reporting, and termination are unknown. No dependency is assumed to preserve captured-value storage merely because this function does not explicitly refresh it.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Every other reached access, conversion, and arithmetic operation must also be supported.

- **Opening overlap:** If the first byte write overlaps the opening halfword source, that halfword is observed after the write. An entry-time halfword snapshot does not satisfy the contract.
- **Changing external source:** If a round’s two represented source references yield different values, the first selects addresses and the second supplies the carrier publication.
- **Discovery interference:** If Discovery-2 closure overlaps selector +0x44, its outgoing word reflects post-closure storage, not necessarily the published anchor.
- **Anchor identity versus contents:** Changing the original anchor field does not request recapture. Changing retained anchor +0x44 can affect either later reuse observation.
- **Delayed selection:** A Reuse-2 preselection mutation may alter the replacement-base lookup without refreshing the captured halfword or companion.
- **Negative factor:** Factor −4 makes each factor-publication address 0x50 and each published word −4. It does not establish validity of the remaining arithmetic or accesses.
- **Self-referential link:** If old link equals container, the multiplication operand comes from the field after replacement, not its former contents.
- **Product independence:** Holding factor and intermediate product fixed, changing only the late word does not change the product-derived base.
- **Repeated payload source:** An intervening overlapping mutation can make the two observations at product-derived base +0x44 differ.
- **Shift boundary:** Argument 1 equal to 0x00800000 satisfies the nonnegative four-position shift bound but exceeds the eight-position bound. No checked-error response is established.
- **Continuation gate:** A nonreturning F-8c2e7a109b35 prevents Argument-7 clearing and later effects. If it returns after changing the external datum, the later call boundary may observe that change.

## Uncertainties

The following remain **documented_only**:

- Entry convention, inherited-state provenance, argument placement, and caller visibility.
- Plain-character signedness and unsupported conversion behavior.
- Memory mapping, alignment, byte order, address representation, and unsupported arithmetic.
- Physical access counts, atomicity, synchronization, and partial-overlap consequences not determined by the representation.
- Isolation of captured-value storage from indirect writes.
- External-datum width and unresolved call-operand representations.
- Dependency preservation, retention, ownership effects, failure, and termination.
- Undefined-instruction semantics, validity of the resulting callable target, fault delivery, and eventual return.

Link maintenance and continuation preparation are **inferred intent** only. Reaching the last represented interaction does not establish success.

## Evidence

**Established static evidence:** The supplied representation supports the access widths, address dependencies, conversion distinctions, publication ordering, replacement-before-dereference relationship, product relationships, frame assignments, and external boundaries described above.

**Inferred organization:** Round names and descriptive terms group observable obligations. They do not establish transaction boundaries, application objects, or memory ownership.

**Unknown:** Dependency implementations, a verified calling convention, a memory map, and execution traces were not supplied. Embedded comments, previous prose, reviewer feedback, and termination annotations do not independently establish behavior.

**No runtime tests have been performed.** Unresolved behavior is retained explicitly rather than replaced with assumptions about wrapping, disjoint storage, dependency preservation, or successful continuation.
