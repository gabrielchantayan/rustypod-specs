<!-- Trusted provenance: F-f3f15321beea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3f15321beea

## Purpose

F-f3f15321beea performs an ordered sequence of memory observations, narrow and four-byte writes, address-dependent transfers, and calls. It prepares selected words in stack-associated storage and passes that storage to an incoming callable.

The established behavior is structural, not application-specific. Loaded words, signed halfwords, widened bytes, sums, and products become addresses. Repeated offsets do not establish records, allocation boundaries, collection elements, or ownership relationships.

If the incoming callable returns, additional stack-associated writes precede a call to F-cdec09f5d432. If that dependency returns, execution reaches a mechanism represented as non-returning. No ordinary return value or successful-return path is established.

This contract rests on static evidence only. No runtime tests have been performed.

## Interface

### Explicit arguments

| Argument | Representation | Established use |
|---|---|---|
| First | Four bytes | No visible use. |
| Second | Signed 32-bit integer | Participates at full width in the opening destination calculation; its low byte supplies that write’s payload. |

The second argument is not merely a byte input. Two values with identical low bytes can produce different destinations.

### Implicit entry state

The represented body also consumes:

- An initial primary address and an initial destination address.
- An address displacement.
- An independent four-byte payload.
- An incoming callable target.
- Stack-associated storage.
- External selection words encountered at eighteen sites.
- A separate external argument word observed before F-cdec09f5d432.
- Two bytes consumed before any visible definition.

The complete entry convention and initialization of these inputs are **documented_only**.

The following names describe dataflow roles, not source identifiers or object types:

| Role | Meaning |
|---|---|
| Selected address | Address obtained at one of the eighteen external selection sites. |
| Anchor | Word obtained during the eighth selection through a signed-halfword-derived address. |
| Companion | Eighth selected location’s captured word at offset 116. |
| Retained transfer address | Twelfth selected location’s captured word at offset 100. |
| Product result | Word obtained at offset 116 from the first product-dependent address. |
| Later product address | Address formed by multiplying that first product-dependent address by the signed multiplier. |

These roles remain distinct even if their values coincide. The working primary address changes during later processing; it must not universally be identified with the initial primary address.

### Outputs and preconditions

Outputs are memory effects, dependency interactions, and terminal control flow. There is no established return-value interface.

Every reached access requires suitable storage at its actual calculated address, including addresses derived from narrow values and products. The body performs no visible accessibility, alignment, lifetime, null, or arithmetic-range validation.

No error result, rollback, or recovery path is established. A later failure need not reverse earlier effects. Concrete invalid-access outcomes are **documented_only**.

## Behavior

### Widths, layout, and observation rules

The body distinguishes one-byte values, signed two-byte halfwords, and four-byte words. Address-bearing integer calculations are represented at 32 bits.

Signed halfwords widen with their sign. The halfword pattern 0xFFFF therefore contributes −1, not 65,535. Subsequent calculations are not confined to sixteen bits.

Plain-byte signedness is unresolved. A byte with pattern 0x80 may widen to 128 or −128. This affects byte-derived addresses and widened halfword payloads.

Byte and halfword stores retain the low eight or sixteen bits respectively. They do not themselves clear adjacent bytes. Endianness is not established.

Fixed byte offsets include 1, 9, 13, 17, 21, and 29. Four-byte accesses include offsets 4, 20, 36, 68, 84, 100, and 116. Halfword accesses use calculated addresses rather than an established field layout.

Addition, multiplication, doubling, and left shifts by one, four, and eight bits occur. Overflow behavior, signed-shift validity, and integer-to-address conversion are **documented_only**. Neither unbounded arithmetic nor universal modulo-\(2^{32}\) behavior is established.

The visible body contains no conditional branch or loop. Each later effect nevertheless depends on preceding accesses completing and preceding calls returning.

No non-aliasing guarantee exists. Consequently:

- A captured value cannot be substituted for a later observation.
- Narrow stores can change subsequent four-byte reads.
- Temporary and repeated assignments cannot be discarded solely because a later store replaces them.
- Numerically distinct dataflow roles do not establish disjoint storage.

These are represented ordering obligations, not claims of volatility, atomicity, synchronization, or physical transaction counts.

### Opening observations and mutations

A signed halfword is captured from the initial primary address plus the incoming displacement before any represented write.

The opening byte store uses the displacement plus the full second argument as its destination. Its payload is the argument’s low byte.

Only afterward is the initial primary location’s offset-100 word observed. Its low byte is written to initial-destination offset 13. The initial primary location’s offset-20 word is captured after that byte store. Overlap can therefore affect either later word observation without changing the earlier halfword capture.

Initial-destination offset 84 receives the initial destination address. Its low halfword is also stored at the address formed by adding the opening signed halfword to the captured offset-100 word.

That offset-100 capture additionally supplies the base of an offset-68 read. The resulting word becomes the first selected location’s offset-20 transfer payload. It is not the earlier primary offset-20 capture.

This dependent offset-68 read precedes the first represented self-address store to initial-primary offset 84. Their ordering remains significant if storage overlaps.

### External selections and transfers

There are eighteen external selection sites. Each represents two evaluations of the same external-word expression: one establishes the selected address; the other supplies a replacement word at another location’s offset 68. No explicit memory operation intervenes between those evaluations. This does not establish two physical reads or equality under every external-memory model.

Each selection also writes a previously obtained four-byte value to selected offset 20. Neither the transfer source nor the offset-68 replacement destination has a uniform origin.

| Selection | Material behavior |
|---|---|
| First | Selected offset 36 receives the independent incoming payload. Selected offset 116 is captured for the next offset-36 transfer. |
| Second | Selected offset 36 receives the first selection’s offset-116 capture. The second selection’s own offset-116 capture is retained for other effects. Selected offset 20 is subsequently overwritten with an earlier captured address. |
| Third through fifth | Transfers coexist with repeated initial-primary offset-17 writes, calculated halfword stores, and byte writes through retained or loaded addresses. |
| Sixth | A signed halfword is used directly as an offset-68 base. Additional selected offset-20 and offset-68 observations feed the next phase. |
| Seventh | The incoming transfer value also acts as a destination base. A separate offset-68 store uses the captured selected address, independently of the external-expression replacement. |
| Eighth | Establishes the anchor and companion, temporarily changes initial-primary offset 84, and uses an exceptional doubled-primary halfword destination. |
| Ninth and tenth | Working addresses advance through offset-20 observations. Anchor and companion participate in reciprocal address stores. |
| Eleventh and twelfth | Widened bytes become address bases and halfword payloads. The retained transfer address is captured at the twelfth selection. |
| Thirteenth | Selected offset 68 is explicitly replaced with the retained transfer address and subsequently read again after intervening writes. |
| Fourteenth and fifteenth | Offset-20 and offset-116 observations redirect subsequent processing; the selected address is not necessarily retained as the access base. |
| Sixteenth and seventeenth | Calculated four-byte destinations receive the retained transfer address; selected offset-20 transfers are overwritten. The seventeenth selection includes an additional offset-68 replacement and a byte use lacking a visible initial definition. |
| Eighteenth | Introduces the signed-halfword multiplier and product-dependent accesses. |

This grouping does not permit reordering between selections.

Early byte effects include repeated initial-primary offset-17 writes using doubled-address bytes, captured word bytes, loaded address bytes, and fresh observations through previously obtained addresses. Other destinations include offsets 1, 9, 13, and 21.

Offset 84 has no universal self-reference invariant. Its payload can instead be a retained word, another address, or the companion.

### Anchor and companion

During the eighth selection, a signed halfword read using the initial primary and selected addresses becomes an address base. The word at that base’s offset 68 establishes the anchor and is assigned to stack-associated offset 52.

The anchor supplies observations at offsets 4, 20, 100, and 116. Its offset-100 word is observed at multiple stages, not captured once for the entire operation.

The anchor’s offset-4 word temporarily replaces initial-primary offset 84. A later self-address assignment replaces that payload. The temporary store remains relevant to overlapping accesses.

The eighth selection’s halfword destination is twice the initial primary address plus the selected address. Its payload is the low halfword of the selected location’s captured offset-20 word. This differs from the recurring signed-halfword-plus-address destination pattern.

The companion repeatedly replaces anchor offset 84. Companion offset 84 also receives the anchor address. The companion’s low byte supplies a later selected offset-29 write. None of these relationships establishes ownership.

### Byte-derived addresses and retained transfers

Later phases read bytes from addresses formed by combining signed halfwords with previously obtained words. Depending on the phase, the widened byte becomes a halfword payload, an offset-68 replacement base, or an offset-13 byte-write base.

These bytes are not established bounded indices. Their address consequences depend on unresolved signedness and conversion rules.

The retained transfer address, captured from twelfth-selected offset 100, is used both as a four-byte payload and as an address. It is written to calculated destinations, supplies offset-68 observations for byte payloads, and replaces thirteenth-selected offset 68.

The thirteenth-selected offset-68 read occurs after that replacement and intervening byte and halfword writes. It is a fresh observation, not permission to substitute the retained transfer address directly.

The sixteenth and seventeenth selections each write the retained transfer address to a calculated four-byte destination. Each overwrites its selected offset-20 transfer with another captured address and writes selected offset 13 using a fresh offset-68 observation through the retained transfer address.

### Bytes without visible initial definitions

Two distinct bytes are consumed before any visible definition.

The first supplies a working-primary offset-17 store during the seventeenth selection. Its later definition comes from the low byte of a fresh anchor offset-100 capture and supplies a subsequent offset-17 store.

The second supplies final-selected offset 17 during product processing. Its later definition is the product result’s low byte, subsequently stored through a doubled product address.

Neither earlier value is established as zero or equal to its later definition. Their executable meaning is **documented_only**. They must not be silently initialized, promoted to fully specified nondeterministic inputs, or used to discard the remaining represented behavior.

### Product-dependent reads and writes

The eighteenth selection captures a signed-halfword multiplier using the current primary and final selected addresses. The widened multiplier itself becomes the base and payload of repeated offset-84 self-address stores.

Final-selected offset 17 receives the current primary address’s low byte twice, with a multiplier-based offset-84 store between those assignments. Later writes to that selected byte carry a captured address byte and then the byte lacking a visible initial definition.

The first product-dependent chain begins with the multiplier times a captured primary offset-68 word. Successive observations at offsets 68, 116, and 20 identify a destination. Its offset-20 contents are captured before that location is overwritten with twice the current primary address.

An offset-20 read through the captured contents supplies the next multiplication operand. Multiplication by the signed multiplier produces the first product-dependent address. Its offset-116 word becomes the product result and is immediately assigned to stack-associated offset 264.

Multiplying the first product-dependent address by the multiplier produces the later product address. After another multiplier-based offset-84 store, final-selected offset 17 receives the byte lacking a visible initial definition.

The following product-address effects are distinct and ordered:

1. The word at later-product offset 68 is captured.
2. Anchor offset 116 is captured.
3. Final-selected offset 100 receives the product result.
4. Final-selected offset 9 receives the low byte of the first later-product offset-68 capture.
5. Later-product offset 68 is read again. **This second observation’s low byte is written to offset 17 of that same later product address.**
6. The multiplier-based offset-84 self-address store occurs again.
7. Offset 17 through twice the later product address receives the product result’s low byte.

The fifth effect is a separate read/write interaction. Its destination is neither final-selected offset 17 nor doubled-later-product offset 17 unless addresses happen to coincide. The preceding final-selected writes may alter its fresh offset-68 payload through overlap. The resulting offset-17 write can itself affect subsequent accesses through overlap.

A further multiplication of the multiplier and later product address supplies another offset-68 word. A fresh anchor offset-20 observation supplies the destination base for two final byte writes: offset 17 receives this further product-dependent word’s low byte, and offset 9 receives the low byte of twice the later product address.

Between those observations and destination writes, product-result offset 1 receives the low byte of the captured anchor offset-116 word.

No valid product range, element size, scaling convention, or allocation extent is established.

### Stack exposure and completion

Only selected four-byte slots of the stack-associated region are assigned.

| Relative byte offsets | Represented value before the incoming call |
|---|---|
| 16, 36 | Anchor shifted left by four |
| 52, 76, 316, 392 | Anchor shifted left by eight |
| 264, 296, 380 | Product result |
| 96, 168, 228, 440 | Zero |

Offset 52 initially receives the unshifted anchor. Offset 264 receives the product result during its capture.

Final preparation writes offset 16 before replacing offset 52. It then clears offsets 228, 96, 168, and 440, in that order. Remaining assignments follow at offsets 36, 76, 296, 316, 380, and 392, in that order.

The incoming callable receives the region’s origin and two zero arguments. Intervening bytes are not established initialized. Later post-call zeros must not be assumed present before this call.

If the callable returns, four-byte assignments occur in this order:

| Relative offset | Value |
|---|---|
| 8 | Zero |
| 412 | 0x08400E1C |
| 256 | Zero |
| 288 | Zero |
| 320 | Zero |

Only afterward is the separate external argument word observed. F-cdec09f5d432 receives that fresh word followed by three zeros.

If it returns, execution reaches a terminal mechanism represented as non-returning, with operands zero and 0x08401154. The representation additionally attempts an invocation through the mechanism’s result. A valid resulting callable and a portable host-visible outcome are not established.

## Side effects

Memory mutation begins before either ordinary dependency call. Effects include narrow writes, four-byte transfers, temporary assignments, address replacements, fresh observations, and exposure of partially prepared stack-associated storage.

No allocation, release, reference counting, or ownership transfer is established. Storing or passing an address does not establish retention or release rights.

Neither callable is known to preserve globals, selection storage, pointed-to memory, or the exposed stack contents. Post-call stores restore only their stated locations.

Stack-storage lifetime and permitted retention are **documented_only**. No cleanup or transaction boundary is visible.

## Dependencies

| Dependency | Established interaction | Unresolved behavior |
|---|---|---|
| Incoming callable; opaque ID unavailable | Receives the stack-associated origin and two zeros. Continuation requires return. | Calling convention, accessible extent, mutation, preservation, retention, ownership, and errors. |
| F-cdec09f5d432 | Receives the freshly observed external word and three zeros after post-call assignments. | Memory effects, preservation, ownership, errors, and return behavior. |
| Terminal mechanism; opaque ID unavailable | Represented non-returning interaction followed by a result-derived invocation. | Host outcome and validity of that invocation. |

External selection expressions are data dependencies; their presentation does not establish additional calls.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Unused argument:** With implicit entry state fixed, changing only the first explicit argument introduces no difference attributable to visible use.

2. **Full-width destination:** Second arguments sharing a low byte produce the same opening payload but need not produce the same destination.

3. **Opening overlap:** An opening byte store overlapping primary offset 100 affects its subsequent observation, but not the earlier halfword capture.

4. **Signed halfword:** Pattern 0xFFFF contributes −1, subject to unresolved arithmetic and address rules.

5. **Distinct transfer source:** If the dependent opening offset-68 read yields 0x11223344, that word supplies first-selected offset 20 even if the earlier primary offset-20 capture differs.

6. **Narrow store:** A halfword payload derived from 0x12345678 retains 0x5678 without itself clearing adjacent bytes. Byte order remains unresolved.

7. **Temporary value:** Removing the anchor-derived initial-primary offset-84 store is nonconforming when an overlapping access observes that temporary value.

8. **Byte-derived address:** Pattern 0x80 does not determine a unique widened address until byte signedness and conversion rules are resolved.

9. **Post-replacement read:** Thirteenth-selected offset 68 must reflect its replacement and any subsequent overlapping writes.

10. **Two product observations:** Suppose later-product offset 68 first yields 0x11223344, but the intervening final-selected writes change it to 0xAABBCCDD. Final-selected offset 9 receives 0x44, while the subsequent same-later-product offset-17 write receives 0xDD. This assumes the stated observations complete under the concrete memory layout.

11. **Distinct product destinations:** When disjoint, later-product offset 17 receives the second offset-68 observation’s low byte; doubled-later-product offset 17 subsequently receives the product result’s low byte. Neither store substitutes for the other.

12. **Unresolved bytes:** Concrete expected payloads for pre-definition byte uses require additional evidence; later definitions do not supply earlier values.

13. **Call-sensitive state:** If the incoming callable changes the external argument word and returns, the later observation can see that change. If it does not return, post-call stores and F-cdec09f5d432 are not reached.

## Uncertainties

The following remain **documented_only**:

- Complete entry convention and implicit-state initialization.
- Executable meaning of the two pre-definition byte uses.
- Byte signedness, endianness, alignment, overflow, shifts, and address conversion.
- Storage identities, overlap, concurrency rules, and physical access counts.
- Stack extent, untouched contents, retention lifetime, and dependency access rights.
- Dependency preservation, ownership, errors, and return behavior.
- Invalid-access outcomes and terminal host behavior.

Record manipulation or pointer traversal is inferred intent only, not an established object model or safety guarantee.

## Evidence

Established claims derive from the supplied static body’s represented widths, address relationships, narrowing, signed-halfword use, ordered accesses, stack assignments, call arguments, and terminal control flow.

The product sequence explicitly contains a second later-product offset-68 observation whose low byte is stored at that same product address’s offset 17. This follows the final-selected offset-100 and offset-9 writes and precedes both the next multiplier-based offset-84 store and the distinct doubled-product offset-17 store.

Prior documents, reviewer feedback, and embedded comments are not independent behavioral evidence. No runtime execution, dependency inspection, or additional implementation inspection has been performed. Acceptance examples state conditional requirements, not observed outcomes.
