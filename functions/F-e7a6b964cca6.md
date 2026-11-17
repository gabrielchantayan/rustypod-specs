<!-- Trusted provenance: F-e7a6b964cca6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7a6b964cca6

## Purpose

F-e7a6b964cca6 executes a long sequence of indirect invocations, derives values from incoming machine state and dependency results, performs ordered mixed-width memory mutations, and ends with an unresolved four-argument indirect invocation.

The supplied representation establishes no application-level purpose. It does not justify interpreting the operation as initialization, serialization, formatting, allocation, or container maintenance. Character-sized constants do not establish a text-output interface.

This contract distinguishes:

- **Established static evidence:** accesses, widths, data dependencies, and ordering visible in the supplied representation.
- **[INFERENCE]:** conclusions requiring an additional machine-arithmetic or execution-model assumption.
- **documented_only:** unresolved behavior for which this document supplies no executable guarantee.

No runtime tests have been performed.

## Interface

### Inputs

The nominal interface has no explicit parameters. Effective inputs nevertheless include:

- An incoming control-flow value used to form the first indirect destination by adding 4094.
- An incoming 32-bit register value used in early shift transformations.
- Primary and secondary values produced at indirect-call boundaries.
- Memory read through derived addresses.
- Fixed-location bytes and words, including a fixed-location word later retained as a publication base.

Secondary results have no established ordinary result declaration or calling convention. They cannot safely be replaced with zero or assumed preserved across intervening calls.

### Outputs and completion

No explicit scalar result is established. Observable outputs are dependency effects, direct memory writes, and the arguments delivered to the final indirect destination.

The representation places a return after that final invocation. Whether the underlying transfer actually returns, behaves as a tail transfer, or follows another control-flow convention is **documented_only**.

There is no visible status code, validation failure result, rollback, or recovery interface.

### Widths and address interpretation

| Quantity | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access and ordinary integer values | 32 bits |
| Selected composite dependency results | 64 bits |

Byte stores retain the low eight bits. Unsigned byte reads widened to words yield values from 0 through 255. Selected composite results supply separate low and high 32-bit halves; other uses extract a value after an unsigned 64-bit right shift by 34, 36, or 44. Those operations are not interchangeable.

Offsets in this document are byte offsets. They describe accessed locations, not established object members. Endianness, alignment handling, address wrapping, memory mapping, and executable-address encoding are **documented_only**.

### Preconditions

Every reached indirect destination must support the represented invocation. Every reached memory location must support its access width and mode.

The function visibly establishes none of these conditions. Nullness, bounds, alignment, lifetime, and access permissions are not checked. Small or fixed addresses must not be silently reinterpreted as valid heap storage.

## Behavior

### Indirect execution and arithmetic

A long ordered progression of indirect calls precedes the direct-memory phase. The first destination depends on incoming control-flow state; later destinations predominantly use code-associated values plus 4094.

Calls have zero through four explicitly represented arguments. Results may be ignored, forwarded, shifted, masked, or retained for later address calculation. Zero explicit arguments do not imply zero implicit inputs. Ignored results do not imply ignorable effects.

The represented progression has no visible input-dependent branch. A dependency may nevertheless fail, diverge, mutate memory, or not return.

Argument transformations include repeated left shifts, forwarding composite-result halves, reusing one transformed value in multiple positions, unsigned right shifts, and masks retaining fields of various widths, including 11–24 bits. Explicit zeros and values derived from explicitly initialized zero state also occur.

Signed left shifts and signed multiplication do not establish portable overflow behavior.

**[INFERENCE]** Under a 32-bit machine-word model retaining the low 32 bits:

- Successive left shifts by 25 and seven yield zero.
- Successive left shifts by 14 and 23 yield zero.
- The low 16 bits of a word shifted left by 16 are zero.
- A value limited to 21 significant bits becomes zero when shifted right by 23 or 24.

These conditional simplifications do not establish the semantics of overflowing signed expressions. Explicit unsigned masking and right shifting have the represented bit-field meaning.

### Labels for retained values

The following are descriptive contract labels, not implementation identifiers:

| Label | Meaning |
|---|---|
| **Anchor** | A late dependency result multiplied by \(2^{25}\) |
| **Selector** | Another result shifted left by eight, masked to 15 bits, then shifted right by six |
| **Small base** | Twice Selector |
| **Payload** | An earlier dependency result shifted left by eight |
| **Tagged base** | A retained composite-result low half shifted left by 25 |
| **Final pair** | The final composite result before direct mutation |

Selector ranges from 0 through 508 in multiples of four, subject to the preceding shift interpretation. Small base consequently ranges from 0 through 1016 in multiples of eight. These ranges establish neither allocation nor valid indexing.

### Initial publication and clearing

The first word publication places Tagged base at offset 20 from Final pair’s high half. A byte at Tagged base plus 9 then receives the low byte of that high half. Anchor is published at offset 84 from a base formed by multiplying Final pair’s low half by 256. The byte at Tagged base plus 9 is subsequently overwritten with Tagged base’s low byte.

The clearing region preserves this order: clear Anchor plus 20; read the byte at Anchor plus 5; clear Small base plus 20; publish the widened captured byte at Anchor plus 52; read the word at Anchor plus 20; clear Small base plus 20 again. Small base plus 21 then receives two byte stores, both using the low byte of that captured word.

The post-clear word read is not unconditionally zero. Intervening accesses can overlap, and ordinary RAM behavior is not established for every location.

### Fixed-address and pointer-derived mutations

A byte read from fixed address 5 is widened and used as an address base. Fixed address 1 receives the low byte of Small base. The byte-derived base plus 21 receives Selector’s low byte.

Payload supplies both a byte at Anchor plus 17 and a word at offset 84 from the captured post-clear word. Anchor plus 17 is then cleared with a byte store.

A word is captured from the byte-derived base plus 84 before that location is replaced with the earlier post-clear word. Small base plus 52 receives Selector.

Words reached at offset 116 from Anchor and Small base supply further destinations at offset 84. One expression uses two reads of Small base plus 116, using a loaded value as both destination-base information and stored data. A single cached read must not be assumed; the representation does not resolve the evaluation order of those two operand reads.

Call the word captured from the byte-derived base plus 84 **Captured base**. Its offset-36 location receives its own base value twice, with intervening publications elsewhere. Its low byte is also stored twice at offset 9 from the address loaded through Anchor plus 116.

Small base plus 20 receives an ordered succession: a shifted dependency-derived word, then Captured base twice. Fixed address 84 receives Captured base doubled. Fixed address 21 later receives the low byte of Small base.

### Scaled destinations and repeated observations

Further destinations derive from Captured base doubled, quadrupled, and multiplied by eight. Byte accesses use offsets including 5, 9, 13, and 21; word accesses use offsets including 4, 20, 52, 68, 84, 100, and 116.

Important read dependencies include:

- A word at Small base plus 116, subsequently doubled.
- A byte at offset 5 from the quadrupled Captured base, subsequently used as a base for a word read at offset 116.
- A word at offset 100 from the quadrupled base.
- A word at offset 4 from that base, subsequently used for byte reads at offset 5.

The last byte location is read more than once, across intervening mutations. These reads are separate observations, not a guaranteed stable byte.

Several fields receive their own base or a related scaled value. This establishes the stores, not linked-object semantics, ownership metadata, or structural invariants.

For the following regions:

- **Quad base** means four times Captured base.
- **Adjusted value** means Quad base minus 150.
- **Adjusted base** means four times Adjusted value.
- **Loaded scale** means the word obtained through the later byte-derived offset-116 read.
- **Doubled base** means twice Loaded scale.
- **Publication value** means four times Loaded scale.

### Constants and publication-base capture

Loaded scale determines three destinations receiving numeric value 58: a byte at four times Loaded scale, a halfword at eight times Loaded scale, and a word at offset 32 from four times Loaded scale.

Adjusted base plus 32 receives, in order, word values 32 and 47, a word containing the zero-extended byte read from Adjusted base plus 4, and word value 32 again.

**The fixed-location word used as Publication base is read next, before the word store of 32 to fixed address 96.** The retained Publication base is therefore the earlier observation. There is no established non-aliasing guarantee between that fixed-location read and address 96, whether through overlapping storage or address-space behavior. The load must not be moved after the store.

### Repeated publication region

A repeated four-access publication pattern follows:

- A word publishes Publication value at offset 84 from an eight-times-scaled destination.
- A byte at Publication base plus 25 receives Adjusted base’s low byte.
- A byte at Adjusted base plus 17 receives Publication base’s low byte.
- A word at Publication base plus 52 receives the eight-times-scaled base used by that publication.

The first group uses eight times Captured base; subsequent groups use eight times Adjusted value. Publication base and the narrowed byte values remain captured values; their source locations are not reloaded for each group.

The groups are explicitly repeated, without a represented loop condition. This contract does not enumerate their exact count; exact multiplicity is **documented_only** here. That limitation does not authorize collapsing them into one assignment or choosing an arbitrary repetition count.

### Final captures and the intervening fixed-address store

After the repeated publications, the function first reads a word from fixed address 169, then a word from Publication value plus 116.

**Only after those reads does fixed address 58 receive a byte derived from eight times Adjusted value plus 48, narrowed to eight bits.** The word at Doubled base plus 20 and the word at Adjusted base plus 4 are captured afterward, in that order.

Thus the address-58 store separates two pairs of captures. If it overlaps either later word location, those later reads can observe its effect. They must not be modeled as snapshots taken before that store.

Call the four captured words, respectively, **Fixed word**, **Transfer word**, **Saved field**, and **Saved offset-four field**.

### Final byte observations and field replacement

Before the next fixed-address publication, byte fields receive narrowed values: Doubled base plus 13 receives Adjusted base; Adjusted base plus 8 receives 108; Adjusted base plus 13 and plus 9 receive Publication base. Doubled base plus 84 receives Doubled base itself.

Fixed address 103 then receives the low byte of Fixed word. **A word is read from Doubled base plus 68 before address 103 is read to supply the byte store to fixed address 33.** The address-33 value is the fresh byte observation, not necessarily the originally published low byte of Fixed word.

The captured offset-68 word supplies a byte at Adjusted base plus 16 and a word at Publication base plus 84. It also supplies a signed right-shift result narrowed to a byte at Doubled base plus 1. The shift count is the low eight bits of Adjusted base. Negative-source behavior and counts of 32 or more remain **documented_only**; count masking is not established.

Publication base plus 84 is then overwritten with Adjusted base. A word from Adjusted base plus 64 is captured before Publication base plus 116 receives Saved offset-four field. The captured offset-64 word replaces the word at Adjusted base plus 16; Publication base plus 68 receives Doubled base.

**Address 103 is read again to supply a byte store to Adjusted base plus 9.** After that store, Doubled base plus 9 receives Adjusted base’s low byte, and Adjusted base plus 68 receives Fixed word as a word. A word is then captured from Adjusted base plus 84.

**Address 103 is read a third time to supply another byte store to Adjusted base plus 9.** Each of the three byte reads from address 103 is distinct. Intervening stores may overlap that address; shared or device-backed storage may also produce different observations.

### Final derived stores and invocation

The word captured from Adjusted base plus 84 becomes **Final base**. Its offset-13 byte receives Transfer word’s low byte. Doubled base plus 84 receives Saved field.

A word captured from Adjusted base plus 20 supplies the later byte at Final base plus 17. Between that capture and that store, Doubled base plus 17 receives Transfer word’s low byte; Adjusted base plus 9 receives Final base’s low byte; and Adjusted base plus 68 receives Final base as a word.

A word read from Doubled base plus 4 becomes **Lookup base**. A halfword value 101 is stored at twice Lookup base, before the word at Lookup base plus 100 is read. Call that later word **Lookup result**.

After the byte publication at Final base plus 17, Final base plus 84 receives word value 100. The word at Lookup base plus 116 is then captured, followed by the word at Doubled base plus 100. Doubled base plus 9 receives Lookup base’s low byte. The latter captured word determines a destination at offset 84 receiving Lookup result; twice Lookup result receives that captured word.

The final indirect invocation receives four represented arguments, in order:

1. Constant 114.
2. Lookup result.
3. Saved field.
4. The word captured from Lookup base plus 116.

No destination identity or result interpretation is established.

## Side effects

Direct writes are not protected by a visible transaction. Failure after earlier stores can leave partial state.

No synchronization, memory barrier, rollback, exception translation, allocation, or release is identifiable in the direct-memory phase. Dependencies may nevertheless allocate, release, mutate globals, or change ownership; no preservation guarantee is established.

Mixed-width accesses may overlap. For example, a byte at offset 21 can overlap a word at offset 20. The resulting word depends on endianness and subsequent observations.

Repeated writes and overwritten values remain part of the represented access sequence. A final-memory snapshot alone is insufficient to establish equivalence when accesses themselves are observable. This contract does not establish that every represented access survives optimization or occurs atomically on an actual execution platform.

## Dependencies

F-e7a6b964cca6 is the only supplied opaque function ID. Its indirect targets have no supplied opaque IDs and remain unresolved; no semantic API names or invented dependency identities are assigned.

Dependency boundaries comprise the entry-state-derived invocation, the subsequent ordered indirect progression, and the final four-argument invocation.

For every unresolved target, argument meaning, primary and secondary result conventions, global preservation, ownership, release behavior, failure behavior, and returning behavior are **documented_only**.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Unsigned byte promotion

If the Anchor-plus-5 byte read observes 254, the subsequent word publication at Anchor plus 52 carries 254, not a sign-extended negative value.

### Selector boundary

Under the stated 32-bit shift interpretation, a selector-source result whose low seven bits are all set yields Selector 508 and Small base 1016. This establishes no validity for Small base plus 116.

### Publication-base read precedes address 96

If overlapping storage or address-space behavior makes the fixed-location load sensitive to the later address-96 store, Publication base still contains the value observed before that store. Reloading afterward is not equivalent.

### Address 58 can affect later captures

If the address-58 byte destination overlaps Doubled base plus 20 or Adjusted base plus 4, the corresponding capture occurs after the byte store. The earlier reads at address 169 and Publication value plus 116 remain earlier observations.

### Address 103 is not a cached source

Suppose the three byte reads at address 103 observe 17, 34, and 51. Fixed address 33 receives 17; the two later source-byte publications to Adjusted base plus 9 receive 34 and 51 in order. Later writes to that destination may overwrite them.

### Width-sensitive constants

For valid destinations, the three publications of 58 use 8-, 16-, and 32-bit accesses. Three byte stores are not equivalent.

### Interrupted dependency

If an earlier indirect dependency does not return, later direct mutations and the final invocation need not occur. No recovery path is visible.

### Final argument order

If execution reaches the final invocation, its represented arguments are 114, Lookup result, Saved field, and the captured Lookup-base-plus-116 word. Successful application-level completion is not implied.

## Uncertainties

The following remain **documented_only**:

- Application purpose and validity of the represented entry point.
- Indirect identities, complete dependency semantics, and secondary-result conventions.
- Signed overflow, address arithmetic, and variable signed-right-shift behavior.
- Endianness, alignment, address mapping, and access atomicity.
- Aliasing, external mutation, and device-backed storage behavior.
- Exact repeated-publication count within this contract.
- Evaluation order of operand reads not ordered by the representation.
- Object layouts, capacities, allocation, ownership, and lifetime.
- Final destination and whether it returns.

These unknowns do not weaken the established read-before-write relationships or permit replacing distinct observations with cached values.

## Evidence

This document uses only the supplied static representation associated with F-e7a6b964cca6. Prior prose and reviewer feedback are not independent evidence.

Static support includes the nominal parameterless interface, indirect invocations, selected 64-bit result handling, mixed-width accesses, fixed-address accesses, retained publication-base load before the address-96 store, address-58 store between the final capture pairs, three distinct byte reads from address 103, and the final four-argument invocation.

Machine-word simplifications are explicitly conditional. No tools, external implementation inspection, runtime execution, or tests were performed. No snapshot hash or independently verified provenance is asserted.
