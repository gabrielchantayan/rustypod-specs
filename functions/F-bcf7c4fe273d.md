<!-- Trusted provenance: F-bcf7c4fe273d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bcf7c4fe273d

## Purpose

F-bcf7c4fe273d performs an ordered, memory-dependent transformation, invokes an incoming indirect dependency, invokes F-cdec09f5d432, and then reaches a represented non-returning terminal transfer.

No application-level purpose is established. Repeated offsets, address-like values, and reciprocal-looking publications do not establish a list operation, allocator, initialization routine, or ownership protocol.

This contract distinguishes:

- **Established static evidence:** accesses, widths, dependencies, and control flow explicitly represented in the supplied material.
- **Inferred intent:** possible interpretations, none of which are acceptance requirements.
- **documented_only:** behavior unresolved without an execution model, complete entry state, or dependency contract.

The seven explicit arguments do not determine execution by themselves. Ambient address-like values, ten ambient data entries, an incoming call target, argument storage, and stack-associated state also participate. No runtime tests have been performed.

## Interface

### Explicit and ambient inputs

| Input | Represented width | Established use |
|---|---:|---|
| Argument 1 | 32 bits | Address base, stored word, and arithmetic operand. |
| Arguments 2 and 3 | 32 bits each | No direct consumption of their values is represented. |
| Argument 4 | 8 bits | Source of the opening byte store. |
| Argument 5 | 32 bits | Its address is passed to the indirect dependency. |
| Argument 6 | 32 bits | No direct consumption of its value is represented. |
| Argument 7 | 32 bits | Assigned zero if the indirect dependency returns. |
| Context A | 32 bits | Incoming ambient address-like value. |
| Context B | 32 bits | Incoming ambient address-like value. |
| Incoming call target | Not independently established | Selects the indirect dependency. |
| Ambient entries 1–10 | 32 bits each | Each is observed twice at its introduction: once to save a base, then again to supply a word publication. |
| Final global input | Not independently established | First explicit argument to F-cdec09f5d432. |

These labels describe roles, not recovered public parameters or object types. The ambient entries are read as data; their association with executable storage does not establish calls to them.

“No direct consumption” does not establish inaccessibility through aliasing or dependencies. Argument 5 has addressable storage. The complete relationship between argument storage, stack-associated storage, and other reachable memory is unknown.

A byte-sized working value is used before any visible initialization. Its provenance is unresolved; it is not established as another argument.

### Memory layout and preconditions

Offsets are byte offsets.

| Access width | Fixed offsets represented | Other addressing |
|---|---|---|
| 8 bits | 1, 9, 13, 17, 21, 29 | Computed sums, including twice a derived base plus 17. |
| 16 bits | No uniform fixed-field layout | Sums involving contexts, ambient-derived bases, signed selectors, and loaded values. |
| 32 bits | 4, 20, 68, 84, 100, 116 | Sums and multiplication-derived addresses. |

No allocation extent, object boundary, alignment, endianness, or address-space interpretation is established. Signed halfwords and widened bytes can become address bases.

Continuation requires each reached access to support its represented width and direction. No explicit null, bounds, alignment, or address-validity checks are shown.

### Output and completion

No ordinary result value or normal-success return is established.

If all preceding operations complete and both ordinary dependencies return, execution reaches the terminal evaluator and its represented non-returning transfer. An earlier fault, termination, or non-return prevents later effects.

## Behavior

### Observation order and aliasing

The behavior is an ordered sequence of observations and writes, not merely a final memory image. Repeated reads remain separate observations. Replaced writes remain observable effects.

Saved values continue to govern their dependent operations despite intervening writes. Conversely, a fresh represented read cannot be replaced by an earlier sample.

No separation is established among argument storage, contexts, ambient entries, computed destinations, or stack-associated storage. A write may therefore affect a later source, address calculation, ambient observation, or dependency-visible location.

The supplied representation does not establish atomicity, volatility, synchronization, concurrency rules, or exact hardware transaction ordering. Correspondence between the represented ordering and a complete execution model is **documented_only**.

### Opening publications

The opening first samples Context A offset 20 as a word, writes Argument 4 to Context B offset 17, samples Argument 1 offset 4, and samples Context A offset 116.

The Argument 1 offset-4 sample is then published as a word at Context B offset 84. Before Context B itself replaces that publication, two intervening operations occur:

- A halfword store at Context A plus twice Context B, using the low sixteen bits of the Context A offset-20 sample.
- A word read at Context A offset 68.

Context B is then written as a word at its own offset 84.

Later, the low byte of the Argument 1 offset-4 sample replaces Context B offset 17. Neither the earlier Argument 4 byte store nor the first offset-84 publication may be omitted merely because a subsequent store replaces it.

### The ten ambient-entry introductions

**Every ambient entry has two separate represented reads.** The first establishes a saved base. The immediately following publication obtains its source word by reading that same ambient entry again, rather than by using the saved base.

For each entry, the second observation supplies a word written at offset 68 of an already determined destination base. Immediately afterward, a previously sampled word is published at offset 20 of the saved base.

There is **no represented write or dependency call between the two ambient reads**. The offset-68 publication occurs after its source observation; its own write cannot explain a difference between the two observations.

Under a sequential interpretation with stable ambient storage, both observations yield the same value. Whether an unspecified execution model permits different observations through external changes or other mechanisms is **documented_only**. Earlier aliasing writes can alter an entry before its pair begins, but do not establish a change within the pair.

The introductions occupy these positions:

| Entry | Position and offset-68 destination |
|---|---|
| 1 | After the opening Context B offset-84 replacement. The destination base is the opening Context A offset-20 sample. |
| 2 | After entry 1’s selector-dependent updates and an offset-84 self-publication. The destination base is the earlier Context B offset-68 sample. |
| 3 | After entry 2’s replacement-base updates and self-publication. The destination base is the word sampled at offset 68 of the then-current base. |
| 4 | After entry 3’s byte-dependent updates, offset-68 read, and self-publication. The destination base is the widened byte obtained during entry 3’s group. |
| 5 | After entry 4’s offset-100 replacement, halfword update, offset-68 read, and self-publication. The destination base is the widened byte obtained during entry 4’s group. |
| 6 | After entry 5’s byte publications, halfword update, fresh offset-68 read, and self-publication. The destination base is the widened byte obtained during entry 5’s group. |
| 7 | After entry 6’s offset-116 replacement, halfword update, offset-68 read, and self-publication. The destination base is entry 6’s current-base offset-68 sample. |
| 8 | After entry 7’s offset-21 publication, offset-116 replacement, halfword update, offset-68 read, and self-publication. The destination base is entry 7’s current-base offset-68 sample. |
| 9 | After entry 8’s selector-based propagation, offset-68 read, and self-publication. The destination base is entry 8’s current-base offset-68 sample. |
| 10 | After entry 9’s selector-based propagation, offset-68 read, and self-publication. The destination base is entry 9’s current-base offset-68 sample. |

For entry 1, the word subsequently published at saved-base offset 20 is the opening Context A offset-68 sample. For entries 2–10, it is the offset-68 sample obtained near the end of the preceding group, before that group’s final self-publication.

Thus, each pair follows the preceding group’s writes and precedes its own offset-20 publication and subsequent byte or selector operations. The entries are not one initial global snapshot.

### Early selector-dependent propagation

After entry 1’s introduction and offset-20 publication, Context B offset 17 receives the saved opening byte. Argument 1 offset 100 is sampled, followed by a Context B offset-68 word and a signed halfword selector addressed using Context B and entry 1’s saved base.

The original Context A offset-116 sample is published at Argument 1 offset 84. A word read through entry 1 offset 20 supplies the base for an offset-13 byte publication and an offset-84 self-publication. Between those publications, the selector contributes to a halfword destination, and entry 1 offset 68 is sampled.

Entry 2’s group again publishes the original Context A offset-116 sample at Argument 1 offset 84. It also uses that original sample as the destination base for a word store of Argument 1 at offset 84.

A significant dependency changes within this group: the selector is sampled using the current base and entry 2’s saved base, but a subsequent word read through an earlier offset-68 sample at offset 20 supplies a replacement address contributor. The later halfword store and offset-68 read use this replacement, not necessarily the base involved in the selector read.

### Byte-derived bases and replacement working value

Entries 3–5 use byte reads addressed by signed selectors plus previously sampled words. Those bytes subsequently supply halfword data or become address bases.

Entry 3’s group writes the low byte of the original Context A offset-116 sample at saved-base offset 29. It also performs a doubled-current-base byte publication, a selector-dependent halfword store, a saved-base offset-68 read, and a self-publication.

Entry 4’s group obtains another byte-derived base. A fresh Argument 1 offset-100 word is published at offset 84 of the signed-selector-derived base. An offset-13 byte publication through the byte-derived base precedes a word read at entry 4’s saved-base offset 100.

That offset-100 word becomes the **later working value**, replacing the original Context A offset-116 sample in that working role. The original and replacement values must not be conflated.

Entry 5’s group samples Argument 1 offset 100 again. The low byte of entry 5’s saved base is written at Argument 1 offset 1. Additional offset-13 publications occur through the saved ambient base and a byte-derived base. The later working value is written at saved-base offset 68 before the selector-dependent halfword store. A subsequent read of that offset-68 field remains fresh and must account for intervening aliases.

### Offset-116 replacements

Entry 6’s group uses a previously sampled offset-68 word to obtain a replacement base through offset 116. The selector was already read before this replacement.

The current base is separately replaced by its offset-20 word. An offset-13 byte publication uses the replacement base’s low byte, while the halfword store uses the replacement base as an address contributor. The replacement’s offset-68 word is then sampled.

Entry 7’s group writes the low byte of its saved ambient base at offset 21 of the preceding offset-68 sample. It also publishes that sample at saved-base offset 68.

A new replacement base is then obtained through offset 116 of the current-base offset-68 sample. The halfword store uses this replacement rather than the ambient base used for the selector read. An offset-68 sample and self-publication finish the group before entry 8’s paired observations.

### Late selector-based propagation

Entries 8 and 9 each use a signed halfword selector itself as the base for an offset-20 word read. Each group includes:

- An offset-84 self-publication through a current-base offset-68 sample.
- A word store of the later working value at the sum of the saved ambient base and a preceding offset-68 sample.
- A publication at saved ambient-base offset 20.
- A fresh full-word read at later-working-value offset 68, supplying an offset-13 byte publication.
- Another offset-13 byte publication through the selector-derived offset-20 base.
- A selector-dependent halfword store, followed by a fresh saved-base offset-68 read and a self-publication.

Entry 9 has additional publications before this shared propagation pattern: a current base is written at offset 84 of a preceding sampled base; a newly sampled Argument 1 offset-100 value supplies a byte at current-base offset 17; and the saved ambient base is written at offset 68 of another loaded base.

Across the central transformation, Argument 1 offset 100 is read **six times**. Its first sample follows entry 1’s initial byte replacement; later samples occur in entries 4, 5, 6, 8, and 9. These are distinct observations. One is published as a full word; others supply later bytes.

### Widths and arithmetic

Selectors are signed 16-bit values. Widening selector bits `0xFFFF` contributes −1, not 65,535.

Byte stores retain the low eight bits; halfword stores retain the low sixteen bits. Address-like values undergo the same truncation when stored narrowly.

Some loaded bytes are character-typed and later widened. Their sign extension or zero extension is **documented_only**. This uncertainty affects addresses and halfword values, not merely presentation.

The represented arithmetic uses 32-bit integer operands, products, doubling, and left shifts by one, four, and eight. Defined behavior for signed overflow, negative left-shift operands, overflowing shifts, and address overflow is not supplied. Neither unbounded arithmetic nor universal modulo-\(2^{32}\) arithmetic is established.

### Multiplication-dependent tail

After entry 10’s paired observations and offset-20 publication, the current base receives the byte retained from the sixth Argument 1 offset-100 sample. Its offset-68 word is then sampled, followed by a signed halfword at the sum of the current base and entry 10’s saved base.

Two identical publication pairs follow. Each writes the signed-halfword-derived base to its own offset 84, then writes the current base’s low byte at ambient-base offset 17. Both pairs are required.

A product of the selector and saved offset-68 word initiates a pointer-like chain through offsets 68, 116, and 20. The resulting base’s offset-20 word is saved before that field is overwritten with the original current base shifted left by one.

The next product uses a word read at offset 20 through the **saved value**, not through the replacement just written. Before the product-derived offset-116 read, another selector-base self-publication occurs, followed by an ambient offset-17 byte store using the saved value’s low byte.

The product-derived offset-116 sample is retained for later publication. Another multiplication by the selector establishes the final product-derived base. A further self-publication follows, then an ambient offset-17 store uses the byte lacking visible initialization.

The remaining order matters:

- Final-product offset 68 and Argument 1 offset 116 are sampled.
- The saved multiplication-phase offset-116 value is published at ambient offset 100.
- The earlier final-product offset-68 sample supplies ambient offset 9.
- A **fresh full-word read** at final-product offset 68 supplies final-product offset 17.
- Another selector-base offset-84 self-publication occurs.
- The saved multiplication-phase value supplies a byte at twice the final-product base plus 17.
- A further word is read at selector times final-product base, plus 68.
- Argument 1 offset 20 is sampled.
- The saved Argument 1 offset-116 byte is written at offset 1 of the multiplication-phase offset-116 value.
- The Argument 1 offset-20 base receives bytes at offsets 17 and 9, using the further word sample and twice the final-product base respectively.

There are five selector-base offset-84 self-publications in this tail.

### Calls and terminal transfer

Before the indirect call, stack-associated assignments establish Argument 1 shifted left by four and eight, followed by four zero words. Subsequent assignments make one additional copy of the shift-by-four value, three additional copies of the shift-by-eight value, and two additional copies of the multiplication-phase offset-116 sample. These assignments do not establish a public stack-frame layout.

The indirect dependency receives Argument 5’s address, zero, and zero.

If it returns, Argument 7 is assigned zero. Further stack-associated assignments establish `0x08400E1C`, then three zero words. The nonzero word is not a zeroing operation or an established return address.

F-cdec09f5d432 then receives the final global input and three zeros. That input is obtained after the indirect call and post-call assignments, not snapshotted at entry.

If this call returns, the terminal evaluator receives zero and `0x08401154`. Its result supplies a represented non-returning transfer target with no explicit arguments.

## Side effects

Writes occur through explicit, ambient, loaded, narrow-value-derived, and arithmetic-derived bases. No bounded output region follows from Argument 1.

Earlier effects may remain visible after a later fault or non-return. No rollback, transaction, compensating action, recoverable error result, or validation protocol is represented.

No allocation or release is directly identified. Dependency ownership, retention, allocation, and release behavior remain unknown. Passing Argument 5’s address does not establish permission to retain it or responsibility for related storage.

Neither ordinary dependency is established to preserve globals, argument storage, ambient state, or reachable memory.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-unknown-1 | Incoming indirect target; receives Argument 5’s address and two zeros. | Identity, complete ABI, mutation, ownership, retention, failure, and return behavior. |
| F-cdec09f5d432 | Receives a freshly obtained global input and three zeros. | Input meaning, effects, preservation, ownership, failure, and return behavior. |
| F-unknown-2 | Terminal evaluator receiving zero and `0x08401154`. | Evaluation semantics and meaning of its result. |
| F-unknown-3 | Target supplied by F-unknown-2; represented as non-returning. | Destination, exception semantics, and recovery possibilities. |

No explicit return value from either ordinary dependency is consumed by subsequent represented operations.

## Acceptance examples

These examples are conditional static requirements, not execution results.

- **Opening replacement:** If the Argument 1 offset-4 sample ends in `0xA7`, Context B offset 17 first receives Argument 4 and later `0xA7`. Keeping only the final store is insufficient.
- **Opening alias:** If the opening halfword destination overlaps Context B offset 84, that halfword write remains between its two word publications, with the intervening Context A offset-68 read preserved.
- **All ten ambient pairs:** For any entry, including entries 3–10, acceptance retains both the saved-base observation and the subsequent observation supplying the offset-68 publication. Substituting the saved value removes a represented observation.
- **Stable ambient storage:** If an entry remains unchanged across its pair under a sequential execution model, both observations agree. No intervening write by this function explains a discrepancy: none is represented between them.
- **Earlier ambient alias:** If the preceding group’s writes alter entry 8, its subsequent paired observations encounter the applicable post-write state. This does not establish a change between those two observations.
- **Signed selector:** Selector bits `0xFFFF` widen to −1. Address validity and surrounding arithmetic remain separate preconditions.
- **Repeated argument input:** If a write changes Argument 1 offset 100 before a later sample, that sample uses the applicable changed state rather than an entry-time snapshot.
- **Saved link:** Overwriting the multiplication-phase offset-20 field does not redirect the next lookup away from its saved former value.
- **Tail reread:** If intervening publications overlap final-product offset 68, its later full-word read may differ from the earlier sample.
- **Uninitialized byte:** No fixed expected byte is established for the store using the value without visible initialization.
- **Dependency mutation:** If F-unknown-1 changes the final global input and returns, F-cdec09f5d432 is not constrained to receive its entry-time value.
- **Completion:** Non-return by either ordinary dependency prevents subsequent operations. If both return and preceding operations complete, the represented path reaches terminal transfer, not an ordinary return.

## Uncertainties

The following remain **documented_only**:

- Application purpose, complete entry state, calling convention, and stack layout.
- Storage overlap, address validity, allocation extents, alignment, and endianness.
- Character signedness and arithmetic outside defined ranges.
- Whether an unspecified execution model permits differing observations within an ambient pair despite no represented intervening write or call.
- Consequences of the byte use without visible initialization.
- Dependency effects, preservation, ownership, lifetimes, and failures.
- Meaning of the fixed post-call word and terminal operands.
- Exact terminal execution, exception behavior, and path completeness.

No portable, deterministic total function of the seven arguments is established.

## Evidence

This contract uses only the supplied static representation. No implementation execution, dependency invocation, runtime trace, arithmetic experiment, or memory-access test was performed.

The representation establishes two ambient-entry references at each of the ten introductions: a saved-base observation followed immediately by a publication whose source references that entry again. It shows no intervening write or call between either pair of observations.

The remaining evidence supports the stated widths, offsets, saved-value dependencies, repeated reads, publications, explicit call arguments, stack-associated assignments, and terminal control flow. Notices about omitted unreachable regions and overlapping global storage limit conclusions about completeness and separation; they establish no additional behavior.

No provenance hash is asserted.
