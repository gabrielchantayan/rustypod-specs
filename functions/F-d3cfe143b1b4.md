<!-- Trusted provenance: F-d3cfe143b1b4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3cfe143b1b4

## Purpose

F-d3cfe143b1b4 has an established **interaction-and-memory contract**, but no established application-level purpose. It participates in an ordered series of external interactions, writes bytes to calculated destinations, obtains two fresh 16-bit memory observations, assigns a retained value to global storage, and writes a 32-bit value through a dependency-supplied pointer.

Conformance depends on access order and value provenance, not merely final memory contents. Two writes remain separate events even when they address the same byte. A primary result and a secondary-channel result remain distinct inputs even when numerically equal.

The global assignment does not establish a reset operation. Receipt of the pointer does not establish allocation, ownership transfer, or permission to release storage.

All obligations concerning later events are conditional on execution reaching those events. Fault-free, portable execution through the entire function is not established.

## Interface

### Effective inputs

There are no explicit entry arguments. The represented behavior nevertheless depends on:

- An incoming eight-bit payload.
- An incoming signed 32-bit arithmetic quantity.
- Incoming control-link information affecting initial dependency selection.
- Implicit execution state, accessible memory, and dependency behavior.

Subsequent interactions provide signed 32-bit controls, one unsigned 32-bit capture, three independent 64-bit results, uninterpreted four-byte results, and secondary-channel observations.

Two secondary observations require particular distinction:

- A pointer later used both numerically and as a 32-bit write destination.
- A signed 32-bit quantity retained for a later byte destination and several outgoing arguments.

Neither may be replaced with the primary result of its producing interaction.

### Outputs and completion

No ordinary return value, success indicator, recoverable-error result, or ownership token is established.

The last represented activity obtains a handling target from F-unknown-03 using immediate zero and numerical location operand 142,643,732, then invokes that target without explicit arguments. Whether this terminates execution, transfers exceptionally, or permits continuation is **documented_only**.

### Widths and interpretation

The direct memory interface comprises eight-bit writes, 16-bit reads, and one 32-bit pointer-directed write. The global assignment’s physical width is unresolved.

Arithmetic controls are signed 32-bit quantities unless expressly described as unsigned or uninterpreted. Throughout this contract:

- **Signed shift scaling** means a signed 32-bit left shift by the stated number of bits. A power-of-two factor describes its scale, not permission to replace it with unrestricted mathematical arithmetic.
- **Signed multiplication** retains signed 32-bit constraints.
- **Unsigned scaling** operates modulo \(2^{32}\).
- **Low-byte projection** retains the least significant eight bits.
- Bit positions are numbered from zero at the least significant bit.

No universal wrapping rule is established for signed overflow. Widening arithmetic can therefore change the contract. Address offset additions also require defined arithmetic.

Character conversion is distinct from low-byte projection: its out-of-range behavior is **documented_only**. Integer-to-pointer, pointer-to-signed-32-bit, and out-of-range signed integer conversions likewise require environmental interpretation.

### Storage and lifetime requirements

Each reached access requires storage permitting its width and any applicable alignment. Each observation requires two readable bytes. The supplied pointer must designate four writable bytes when the later word write occurs.

Disjointness is not required. Calculated destinations, observation regions, global storage, pointer-designated storage, and dependency-accessible memory may overlap.

Pointer lifetime is not guaranteed by receipt. Unknown dependencies may modify globals, modify overlapping memory, or invalidate storage. Their preservation and ownership rules must not be assumed.

## Behavior

### Observable access structure

The byte-write stream has six ordered groups. Occurrence numbers below are local to their group and count writes, not distinct destinations.

| Group | Occurrences | Common payload |
|---|---:|---|
| Entry | 20 | Incoming eight-bit payload |
| Zero | 6 | Zero |
| Address-derived | 1 | Character conversion of the numerical destination |
| Capture | 13 | Retained capture byte |
| Capture field | 1 | Character conversion of capture bits 19–28 |
| Final | 21 | Retained product bits 29–31 |

These groups contain 62 byte-write events. Dependencies also occur before the first event, between events, and after the last event. Their results being unused does not make the interactions removable.

For an **ordinary boundary**, a signed control determines:

- The numerical byte destination by signed shift scaling of three bits, corresponding to a factor of eight.
- Two equal explicit arguments to the immediately following interaction, each by signed shift scaling of fourteen bits, corresponding to a factor of 16,384.

The byte write precedes that interaction. Exceptions below change only the specified aspects of this rule.

No represented data-dependent branch or loop selects a different group sequence. Arithmetic interpretation, inaccessible storage, and nonreturning dependencies can nevertheless prevent later events.

### Retained values and provenance

The following values remain conceptually separate from mutable memory and newer dependency results:

| Retained value | Origin and later significance |
|---|---|
| Entry scale | Ordinary outgoing quantity at entry occurrence 16; later determines capture occurrence 2 and the retained product |
| Capture | Unsigned 32-bit dependency result; supplies capture payloads and the capture-field payload |
| Capture byte | Low-byte projection after unsigned scaling of the capture by eight; shared by all 13 capture writes and the global assignment |
| Product | Signed multiplication of the entry scale by 32,768; supplies capture occurrence 6 and the final payload |
| Product’s highest byte | Highest byte of the product’s unsigned 32-bit interpretation; retained after capture occurrence 6’s outgoing interaction |
| Supplied pointer | Secondary-channel pointer acquired before final occurrence 4 |
| Retained secondary quantity | Signed 32-bit secondary observation acquired after final occurrence 4 |
| Retained word | Signed multiplication of final occurrence 17’s fresh primary control by four |

The capture byte ranges from zero through 248 in steps of eight. The final payload is the highest retained product byte reduced to its top three bits, hence ranges from zero through seven.

The product’s highest byte originates from the product, not from the interaction following capture occurrence 6. Later payloads do not originate from rereading the global assignment. The retained word is not subsequently reconstructed by reading pointer-designated memory.

### Entry payload distribution

All 20 entry writes carry the same incoming byte. Most have ordinary boundaries.

Entry occurrence 4 uses a control returned by F-unknown-02. Its producer receives an earlier primary signed result scaled by a two-bit signed shift. The byte destination uses signed multiplication of the returned control by two. The following interaction receives two copies of that control scaled by a twelve-bit signed shift, corresponding to 4,096.

Entry occurrence 5 has an ordinary destination but four explicit outgoing positions:

| Position | Required value |
|---|---|
| 1, 2, 4 | Ordinary fourteen-bit signed-shift quantity |
| 3 | Incoming arithmetic quantity scaled by a ten-bit signed shift |

The next interaction receives the preceding interaction’s secondary signed result scaled by a three-bit signed shift. Its input is not derived from that interaction’s primary result.

At entry occurrences 6–8 and 11–16, the primary result of the immediately following outgoing interaction is forwarded after a two-bit signed shift. That forwarded result is distinct from the control that selected the byte destination.

The outgoing quantity at occurrence 16 remains the retained entry scale regardless of subsequent primary results.

The first two independent 64-bit acquisitions occur in the intervals after entry occurrences 17 and 18, respectively. Their outgoing layouts are specified under Dependencies.

Entry occurrence 20 has an ordinary destination, but its outgoing equal pair uses signed multiplication of the control by four rather than the ordinary fourteen-bit shift.

### Zero and address-derived writes

All six zero writes have ordinary boundaries. No direct preliminary read of their destinations is represented.

The following address-derived write also has an ordinary destination. Its payload is character conversion of the destination’s numerical value; destination contents are not an input.

Its outgoing interaction receives the ordinary equal pair followed by a third explicit argument containing that numerical destination. Reading destination memory to manufacture the payload would introduce an access and change provenance.

### Capture-byte distribution

The capture group uses its retained byte for all 13 writes. Its nonuniform address and outgoing layouts are:

| Occurrence | Numerical destination | Immediately following explicit arguments |
|---|---|---|
| 1 | Capture unsigned-scaled by sixteen | Capture unsigned-scaled by 32,768, twice |
| 2 | Entry scale, signed shift of 16 bits | Entry scale, signed shift of 27 bits, twice |
| 3–5 | Fresh signed control, signed shift of 13 bits | Same control, signed shift of 24 bits, twice |
| 6 | Product, signed shift of 12 bits | Product, signed shift of 23 bits, twice |
| 7 | Fresh signed control, signed shift of 13 bits | First-observation layout |
| 8–12 | Ordinary destination | Ordinary equal pair |
| 13 | Ordinary destination | Control multiplied signed by four, twice |

Both unsigned calculations at occurrence 1 wrap at 32 bits.

The retained product is established before occurrence 2’s byte write. Its highest byte is retained after occurrence 6’s outgoing interaction returns.

The interval after occurrence 13 additionally includes forwarding a newly acquired signed primary result after a one-bit signed shift.

### First observation and global state

At capture occurrence 7, the byte write precedes a fresh 16-bit read, without an intervening dependency invocation.

The observation address is the signed control multiplied by 4,096, followed by a 56-byte offset. The outgoing interaction receives, in order:

1. The multiplication result without the offset.
2. The newly observed 16-bit quantity.

Aliasing may make the observation depend on earlier direct writes or dependency effects. Reusing an older observation is not equivalent.

After this observation and intervening interactions, but before capture occurrence 8, the retained capture byte is assigned to designated global storage. The immediately following interaction receives two explicit zeros.

The global destination’s type, assignment conversion, and physical write width are **documented_only**. Later dependencies may overwrite it. The assignment therefore provides no unconditional final-global-value guarantee.

### Capture-field write

The capture-field payload begins as the ten-bit quantity represented by capture bits 19–28, with range zero through 1,023, then undergoes character conversion.

The byte destination is ordinary. Its outgoing equal pair uses signed multiplication of the destination control by four.

Only under a low-eight-bit-preserving character conversion rule does the stored byte depend solely on capture bits 19–26. That conversion rule is not independently established.

### Final-byte distribution

All 21 final writes have the same three-bit payload. The later reduction of the retained highest byte does not reduce an already reduced payload again. This continuity extends across the second observation.

Most boundaries are ordinary. The exceptions are:

| Occurrence | Required distinction |
|---|---|
| 2 | Control formation includes a two-bit signed left shift of a fresh primary result, followed by a right shift of 32 bits on the represented signed 32-bit quantity. Conditional on meaningful continuation, destination formation uses signed multiplication by two and outgoing arguments use a twelve-bit signed shift. |
| 4 | The supplied pointer’s signed 32-bit numerical conversion is the basis. Destination and outgoing equal pair use signed shifts of 8 and 19 bits, respectively. |
| 9, 16, 20 | The immediately following interaction has no explicit arguments. |
| 11 | The retained secondary quantity is the basis. Destination and outgoing equal pair use signed shifts of 11 and 22 bits, respectively. |
| 15 | The outgoing interaction uses the second-observation layout. |
| 17 | The byte write precedes the pointer-directed word write; the outgoing interaction receives the retained word in three positions. |

The third independent 64-bit acquisition occurs between final occurrences 2 and 3.

The right shift by 32 is an unresolved execution boundary, not authorization to widen the operand, mask the shift count, or choose a constant result.

### Pointer use and retained secondary state

The pointer is supplied through a secondary result channel before final occurrence 4. Its roles must remain separate:

- Its numerical signed 32-bit conversion contributes to calculated addresses.
- The pointer itself designates the later 32-bit write destination.

Scaling its numerical conversion does not relocate the pointer-directed word destination.

After occurrence 4’s equal-pair interaction, the second subsequent interaction without explicit arguments supplies the retained secondary signed quantity. Newer secondary results do not replace that retained quantity.

Each interval after final occurrences 11–14 contains a two-position transfer: a fresh primary result scaled by a two-bit signed shift, followed by the retained secondary quantity.

### Second observation and word update

At final occurrence 15, an ordinary byte destination is selected by a fresh primary control. The byte write precedes a fresh 16-bit read without an intervening dependency invocation.

The observation address uses the pointer’s signed 32-bit numerical conversion multiplied signed by 128, followed by a 32-byte offset. The next interaction receives only the fresh observation.

For both observations, interpretation as signed or unsigned, byte order, and outgoing argument extension are **documented_only**.

At final occurrence 17, the retained word is formed by signed multiplication of the fresh primary control by four. The ordinary byte write occurs before the 32-bit write through the supplied pointer, with no dependency invocation between the two writes. Only afterward does the outgoing interaction receive three copies of the retained word.

If the destinations overlap, the word write may overwrite the byte. Word byte order, constituent-byte ordering, and atomicity are unresolved.

The following interval includes an uninterpreted four-byte primary result forwarded unchanged into both positions of an equal pair.

Between final occurrences 18 and 19, a two-position transfer receives a newer signed primary result scaled by a two-bit signed shift, followed by the retained word. The second position is not a memory reread.

Further dependency interactions separate the final byte write from terminal handling.

## Side effects

Conditional on reaching all relevant events, the direct access inventory is:

- 62 eight-bit writes.
- Two fresh 16-bit reads.
- One pointer-directed 32-bit write.
- One global assignment of unresolved physical width.

This inventory excludes dependency-internal accesses and is not a specification of bus transactions.

No direct null check, bounds validation, retry, rollback, release, or cleanup protocol is established. A later failure does not establish reversal of earlier effects.

No synchronization, concurrency safety, atomicity, reentrancy guarantee, or final-memory snapshot is established. Retained scalar provenance remains distinct from mutable storage contents.

## Dependencies

Only opaque role identifiers are available:

| ID | Established role |
|---|---|
| F-unknown-01 | Family of external interactions |
| F-unknown-02 | Exceptional entry-control producer |
| F-unknown-03 | Terminal handling-target provider |

F-unknown-01 does not establish one common callable target. Initial selection depends on incoming control-link information; later selections generally depend on embedded control-location information. F-unknown-02 has a separate aligned selection relationship. These relationships do not resolve an architecture or complete calling convention.

### Wide-result argument layouts

Each of the first two 64-bit acquisitions supplies a subsequent three-position transfer:

1. Signed 32-bit conversion of the whole result.
2. Its upper 32-bit portion as an uninterpreted four-byte quantity.
3. The same upper portion again.

The third acquisition supplies two positions: signed 32-bit conversion of the whole result, then signed 32-bit conversion of its upper portion.

Whole-result signed conversion is not unconditionally equivalent to extracting the lower 32 bits. These layouts do not establish memory endianness.

### Other interactions

Distinct secondary four-byte observations are forwarded in the intervals after final occurrences 4–7, 10, 18, and 19. They are neither primary results nor substitutes for the retained secondary signed quantity.

The interaction history also contains unchanged-primary-result forwarding, signed-primary-result forwarding after two-bit signed shifts, and invocations without explicit arguments. The grouped description does not authorize deleting, coalescing, or reordering those interactions.

Resolved identities, implicit inputs, internal effects, preservation rules, ownership effects, and return guarantees remain **documented_only**. A complete ABI-level interaction trace is not established by this contract.

## Acceptance examples

These are conditional criteria, not executed tests. Each assumes local reachability, valid storage, defined arithmetic and conversions, and returning dependencies as needed. They do not assert a jointly realizable whole-function execution.

1. **Ordinary ordering.** Entry payload 165 and ordinary control three require a byte write of 165 at numerical address 24 before the outgoing pair 49,152 and 49,152. Matching final memory after reversing that order is insufficient.

2. **Exceptional entry boundary.** F-unknown-02 returning three requires entry occurrence 4’s destination six and outgoing pair 12,288 twice. Its producer input must retain the specified earlier-primary-result provenance.

3. **Secondary-channel distinction.** Entry occurrence 5 with control three and incoming arithmetic quantity two requires outgoing positions 49,152, 49,152, 2,048, and 49,152. A secondary signed result of five then requires 40 at the next interaction, independently of the primary result.

4. **Unsigned wrapping.** Capture 35 requires capture byte 24, first capture destination 560, and outgoing pair 1,146,880 twice. Capture 268,435,457 requires byte eight, destination sixteen, and outgoing pair 32,768 twice.

5. **Different capture projections.** Capture 524,288 requires capture byte zero but capture-field quantity one before character conversion. Changing only bits 27–28 preserves the field byte only under low-eight-bit-preserving conversion.

6. **No destination-content dependency.** Address-derived control three requires character conversion of numerical address 24, regardless of the preexisting byte stored there.

7. **Fresh first observation.** Capture occurrence 7 with control two requires the byte write at 16,384 before a 16-bit read at 8,248. The outgoing arguments are 8,192 and the actual fresh observation.

8. **Pointer identity and overlap.** Pointer numerical conversion sixteen requires final occurrence 4’s destination 4,096 and outgoing pair 8,388,608 twice. The second observation address is 2,080. If the pointer itself designates address sixteen and occurrence 17’s control is two, the byte write at sixteen precedes a word write of eight there, followed by the outgoing triple eight, eight, eight.

9. **Payload continuity.** A locally retained highest product byte of 224 requires final payload seven before and after the second observation. Reducing the three-bit payload again to zero is nonconforming.

10. **Width restrictions.** Under lower-32-bit-preserving signed conversion, a wide result with lower portion three and upper portion two requires early layout three, two, two and third-acquisition layout three, two. Under nonnegative, signed-representable left-shift rules, ordinary control 131,072 permits destination formation but not the outgoing fourteen-bit shift.

## Uncertainties

The following are **documented_only**:

- Application intent, architecture, execution mode, and valid entry conditions.
- Dependency identities, implicit interfaces, preservation, and return behavior.
- Storage accessibility, alignment, aliasing, extent, and lifetime.
- Ownership transfer and release obligations.
- Character, pointer, and out-of-range signed conversions.
- Observation signedness, byte order, and argument extension.
- Global assignment type, conversion, and physical width.
- Fault propagation, concurrency behavior, and terminal continuation.

**Full-width shift:** A right shift by 32 on a represented signed 32-bit quantity has no established portable interpretation, including when the operand is zero. All later obligations remain conditional on environmental interpretation and continued execution.

**[INFERENCE] Strict signed-shift consequence:** If signed left shifts require nonnegative operands and signed-representable results, the retained entry scale is a nonnegative multiple of 16,384. Its later 27-bit shift admits only values zero through fifteen. Together, those constraints admit only zero. This does not establish accessible storage or resolve the full-width shift.

The annotation concerning omitted unreachable material supplies no behavioral evidence for that material.

## Evidence

**Established static evidence:** The supplied representation supports the direct access widths and counts, ordered payload groups, exceptional argument layouts, primary/secondary distinctions, retained-value provenance, fresh-read boundaries, global-assignment placement, byte-before-word ordering, and terminal operands.

**Derived under stated premises:** Payload ranges, capture-field width, unsigned wrapping, numerical acceptance examples, and the strict signed-shift consequence.

**Inferred intent:** No application-level interpretation is asserted.

**Not established:** Valid storage, dependency preservation, ownership guarantees, portable entry-to-completion execution, ordinary completion, or runtime outcomes.

Prior prose, reviewer feedback, and embedded comments are not independent behavioral evidence. No runtime execution or tests have been performed.
