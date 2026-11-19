<!-- Trusted provenance: F-b9d3f895d4bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9d3f895d4bd

## Purpose

F-b9d3f895d4bd specifies an ordered sequence of external interactions and memory effects, followed by a represented non-returning control transfer.

Its application purpose is not established. The evidence does not justify describing it as initialization, buffer clearing, reset handling, or an update to a particular structured object. In particular, separately selected destinations are not established to form a contiguous region.

This contract distinguishes:

- **Established static evidence:** access widths, data provenance, argument relationships, and ordering explicitly supported by the supplied representation.
- **[INFERENCE]:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior requiring platform, dependency, or interface information.

Requirements apply only when execution reaches the relevant boundary. They do not guarantee that earlier dependencies return, arithmetic is defined, or storage remains accessible.

## Interface

### Inputs

The represented interface has four positional inputs.

| Position | Width and interpretation | Established contribution |
|---|---|---|
| First | Signed 32-bit integer | Determines both equal arguments of the initial dispatch, at scale 4,096. |
| Second | Signed 32-bit integer | Determines the first byte destination, at scale two. |
| Third | 32-bit quantity; interpretation unavailable | No explicit use is represented. |
| Fourth | Signed 32-bit integer | Determines the third argument of a later four-argument interaction, at scale 1,024. |

Additional effective inputs include:

- An incoming eight-bit datum used by the initial group of byte writes.
- Incoming continuation-related state used to select the first external target.
- Primary dependency results, represented mainly as 32-bit quantities.
- Four results represented as 64-bit quantities.
- Secondary dependency outputs, including an address-valued output and a separate 32-bit companion.
- Contents of the two locations subsequently read as halfwords.
- Any hidden inputs consumed by external dependencies.

An interaction with no displayed arguments is not established to have no inputs. It may consume implicit register state, memory, globals, or other calling-convention state.

The unused third positional input is not sufficient evidence that the actual interface is independent of all corresponding entry state.

### Outputs and completion

No ordinary result value, status code, success flag, or caller-provided result object is represented.

Observable outputs consist of memory effects, external interactions, one global assignment, and the terminal transfer. The terminal mechanism receives selector zero and a fixed location-associated operand, numerically 142,643,732. Its produced destination is then invoked without displayed arguments. This path is represented as non-returning.

Actual trap handling, termination, handler execution, or other platform behavior is **documented_only**. An ordinary successful return is not established.

### Storage and ownership preconditions

Every reached access requires accessible storage of the specified width at that time. No allocation origin, capacity, containing object, or ownership transfer is established.

Retaining an address numerically does not preserve the referenced storage’s lifetime or mapping. Dependencies are not assumed to preserve storage, globals, mappings, or ownership. Allocation, retention, invalidation, and release by dependencies remain **documented_only**.

## Behavior

### Execution and observation model

The supplied body contains no visible conditional branch, loop, retry, validation path, or ordinary early return. Dependencies can nevertheless interrupt progress by not returning or by otherwise changing the execution environment.

The direct byte effects divide into the following data-provenance groups:

| Data source | Number of represented byte-write events |
|---|---:|
| Incoming eight-bit datum | 22 |
| Literal zero | 6 |
| Byte derived from its own destination quantity | 1 |
| Retained byte derived from an unsigned result | 13 |
| Independently extracted byte from that unsigned result | 1 |
| Three-bit datum derived from a separately retained product | 21 |

These are **64 byte-write events**, not a promise of 64 distinct locations. The global assignment and the direct word write are additional effects.

The groups occur in the listed order. The first halfword read and global assignment occur within the retained-byte portion; the second halfword read and direct word mutation occur within the three-bit portion.

External calls separate many accesses. They may observe earlier effects or alter storage before later effects. Consequently, acceptance concerns the reached event sequence, not merely a final memory image. Calls with unused results remain interactions; apparent repetition does not justify removing or combining them.

### Incoming-data effects

The first direct effect stores the incoming datum as one byte at twice the second positional input’s numeric value. No explicit read, null check, bounds check, or conditional suppression precedes it.

The initial dispatch follows that write. Both displayed arguments are derived from the first positional input at scale 4,096, and its target depends on incoming continuation-related state.

The remaining incoming-data writes use dependency-selected destinations. Most use an eightfold destination and are followed by an interaction receiving two equal values at scale 16,384 from the same selecting result. The exceptions are behaviorally significant:

- The first dependency-selected byte write uses an eightfold destination, but its following interaction has a single argument at scale four and produces the first represented wide result.
- One later destination uses scale two, with duplicated arguments at scale 4,096.
- One eightfold write precedes a four-argument interaction.
- The final incoming-data write uses an eightfold destination with duplicated arguments at scale four.

At the four-argument boundary, the first, second, and fourth arguments agree and use scale 16,384 from the result selecting the preceding destination. The third argument instead uses the fourth positional input at scale 1,024. The next interaction consumes a secondary output at scale eight; the primary result is not a substitute.

Intervening call-only activity includes duplicated primary results, unchanged forwarding, and fourfold forwarding. One target selection depends on four-byte alignment. These interactions do not establish a common external function identity.

### Zero-data effects

Exactly six byte writes have literal zero as their explicit data source. Each destination is selected separately at scale eight from a dependency result. The following associated interaction receives duplicated scale-16,384 arguments from that selecting result.

These writes do not establish a six-byte buffer, adjacent addresses, distinct addresses, or a clearing operation. Other data groups can also happen to contain zero; the count of six refers specifically to literal-zero provenance.

### Derived and retained data

The later byte values have distinct origins:

| Role | Established stored-data relationship |
|---|---|
| Self-derived byte | Lowest eight bits of the eightfold quantity also used as its destination. |
| Retained byte | Lowest eight bits of an unsigned 32-bit dependency result scaled by eight. |
| Independent extraction | Original bits 19–26 of that same unsigned result. |
| Three-bit datum | Bits 29–31 of a separately retained 32-bit product. |

Bit zero is the least significant bit. Equality of resulting values does not make their sources interchangeable.

At the self-derived boundary, the following interaction receives two scale-16,384 arguments and then the full eightfold quantity. Its third argument is not narrowed to the stored byte.

The retained byte is first stored at a destination selected from its unsigned originating result at scale 16. The following arguments duplicate the originating result at scale 32,768. Later retained-byte writes reuse the retained value; they do not reread the first destination.

The independent extraction likewise uses the original unsigned result rather than the retained byte. Although the intermediate selection spans ten bits, only original positions 19–26 survive the byte store. Original positions 27 and 28 do not contribute to that stored byte.

### Retention across intervening interactions

Some accesses use older retained results instead of the latest dependency result.

One retained quantity was already scaled by 16,384 when first supplied as duplicated arguments during the incoming-data portion. Relative to that retained quantity:

- A later retained-byte destination uses scale 65,536.
- Its associated duplicated arguments use scale 134,217,728.
- A separately retained product uses scale 32,768.

The product later selects a retained-byte destination at scale 4,096, followed by duplicated scale-8,388,608 arguments. Its 32-bit pattern also supplies the eventual three-bit datum.

These descriptions preserve intermediate-width dependencies. They do not permit replacing the represented operations with unlimited-precision multiplication or assuming signed wraparound.

Four other retained-byte destinations use fresh results at scale 8,192. The first three have associated duplicated scale-16,777,216 arguments. The product-selected write lies between the third and fourth of these fresh-result writes.

At the fourth fresh-result write, a 16-bit read follows without an intervening dependency call. Its location is selected from the same result at scale 4,096, with an additional 56-byte displacement. The consuming interaction receives the scale-4,096 quantity first and the halfword read second.

### Global assignment and subsequent retained-byte writes

After further interactions, a separately designated global destination is assigned the retained byte. Its declaration, storage width, and application meaning are unavailable.

The assignment precedes an interaction with two literal-zero arguments. Those arguments neither make the assigned value zero nor establish that the dependency preserves the assignment.

Six further fresh, eightfold destinations receive the retained byte before the independent-extraction write:

- The first five associated argument pairs use scale 16,384.
- The sixth pair uses scale four.

The independent-extraction write also uses an eightfold destination and duplicated scale-four arguments.

Between the sixth retained-byte write and the independent-extraction write, one forwarding interaction uses scale two rather than the otherwise frequent scale four.

### Three-bit-data effects

The final byte-data group uses a value between zero and seven, conditional on an established 32-bit pattern for the retained product. Its origin is separate from the unsigned result supplying the retained byte and independent extraction.

Most destinations use fresh results at scale eight, with associated duplicated scale-16,384 arguments. Not all writes have this associated-call shape: three are followed by interactions without displayed arguments, and others participate in the special access boundaries below.

One destination depends on a right shift of 32 positions applied to a signed 32-bit quantity after a left shift. The supplied representation does not establish a valid result for that operation. Its doubled destination and following duplicated scale-4,096 arguments remain **documented_only**. Neither zero nor a processor-specific masked-shift result is an unconditional acceptance requirement.

### Saved secondary outputs

Two secondary outputs have different retention roles.

The **saved address** is used for:

- A byte destination at scale 256.
- Associated duplicated arguments at scale 524,288.
- A later halfword location at scale 128 with a 32-byte displacement.
- A direct 32-bit write through the address itself.

The **saved companion** is a separate 32-bit quantity. It selects a byte destination at scale 2,048, followed by duplicated scale-4,194,304 arguments. It also occupies the unchanged second argument position in four later two-argument interactions. Each corresponding first argument is a fresh result at scale four.

The companion is not the saved address, a current primary result, or the latest secondary output. Its numeric retention is established; the calling-convention mechanism producing and preserving these secondary outputs is not.

### Second halfword read and word mutation

Immediately before the second halfword read, a fresh eightfold destination receives the three-bit datum. No dependency call intervenes between this write and the read.

The read accesses 16 bits at the saved address’s numeric value scaled by 128, plus 32 bytes. Its consuming interaction receives the halfword as its sole displayed argument.

Later, a fresh result determines a paired mutation:

- The three-bit datum is stored at its eightfold destination.
- A 32-bit word equal to four times that result is then written directly through the saved address.

No dependency call separates those writes. The following interaction receives the word value in all three displayed positions. That value is retained for a subsequent two-argument interaction, where it occupies the second position alongside a fresh result at scale four.

The scaled byte destination, displaced halfword location, and direct word destination are separate accessibility requirements. Sharing an originating address-valued output does not establish a structure layout.

Three further byte writes occur after the word mutation, along with additional dependency activity. The word write is neither the final memory effect nor an established completion indicator. The terminal mechanism is reached only after this subsequent activity.

### Arithmetic and representation boundaries

Most arithmetic quantities are represented as 32 bits. Explicit unsigned left shifts retain 32 bits, discarding overflowing high bits. Byte stores retain eight data bits.

Signed multiplication and signed left shifts do not establish wrapping arithmetic. Their scale relationships are conditional on a defined interpretation of the represented operations. Signed overflow, negative operands, address conversion, and the out-of-width shift require additional platform evidence.

Both halfword reads are 16-bit accesses. Byte order, alignment legality, argument extension, atomicity, and synchronization are unresolved. The direct word mutation is a 32-bit access. The global assignment has an eight-bit data source but an unresolved storage width.

## Side effects

Established direct effects are:

- 64 byte-write events with the provenance distribution above.
- Two 16-bit reads.
- One assignment to a separately designated global destination.
- One direct 32-bit write.
- External interactions and the terminal control transfer.

Destinations are not proven disjoint. Overlap preserves event ordering but can change later observed contents. A later write can replace earlier bytes; a read observes the contents available when reached, including changes made by dependencies.

No explicit allocation, deallocation, cleanup, rollback, diagnostic, or transactional mechanism is represented. This does not constrain undisclosed dependency behavior.

Interrupted progress can leave a visible prefix of effects. No represented local action restores earlier memory or translates a dependency failure into a returned error.

## Dependencies

F-b9d3f895d4bd is the only supplied resolved opaque function ID. External dependency IDs and their identity mapping are **documented_only**. Interaction-role descriptions here do not assign identities or imply that equal argument lists designate the same function.

The dependency surface includes entry-state-dependent dispatch, location-associated selection, one alignment-dependent selection, primary results, secondary outputs, and terminal handling.

The four represented wide results have these ordered consumer relationships:

| Wide-result occurrence | Ordered 32-bit portions supplied to its consumer |
|---|---|
| First | Lower, upper, lower |
| Second | Lower, upper, upper |
| Third | Lower, upper, upper |
| Fourth | Lower, upper |

These are bit-portion relationships, not memory layouts. They establish neither endianness nor floating-point interpretation. Whether the actual interface supplies a single 64-bit result or paired outputs is unresolved.

Seven later forwarding interactions consume secondary quantities without an explicit scale. Replacing them with primary results would lose established argument provenance.

No dependency is established to be pure, non-failing, guaranteed to return, memory-preserving, global-preserving, or ownership-preserving.

## Acceptance examples

These are conditional static deductions, not executed tests. Each assumes that its boundary is reached, required storage is accessible, preceding interactions complete, and relevant arithmetic has a defined interpretation. A local example does not prove that a complete execution with those values is realizable.

### Entry and interrupted progress

With second positional input 256 and incoming datum 165, the first effect stores byte 165 at numeric address 512.

With first positional input three, the initial dispatch receives 12,288 in both displayed positions. If that interaction does not return, subsequent effects are not required, and no represented rollback removes the first write.

### Argument provenance

With selecting result two at the four-argument boundary and fourth positional input five, the arguments are **32,768; 32,768; 5,120; 32,768**.

Using 32,768 in the third position would violate its explicit-input provenance. Using the primary result instead of the following scaled secondary output would also violate the established relationship.

### Narrowing and independent extraction

At the self-derived boundary, result 35 selects address 280 and stored byte pattern 24. Its consumer receives **573,440; 573,440; 280**. A third argument of 24 would incorrectly narrow a separately observable value.

At the retained-byte boundary, unsigned result 35 supplies byte 24, initially stored at address 560, with duplicated arguments of 1,146,880. Subsequently changing address 560 does not itself change the retained datum.

Unsigned originating result 524,323 supplies retained byte 24 but independently extracted byte one.

### Global and wide-result behavior

For retained byte 24, the global assignment uses 24 while the following interaction receives two zeros. The six subsequent retained-byte writes also use 24, but neither distinct destinations nor preservation of the global value is guaranteed.

For wide-result lower portion 17 and upper portion 34, the consumers receive, respectively, **17, 34, 17**; **17, 34, 34**; **17, 34, 34**; and **17, 34**.

### Access timing and address reuse

At the first halfword boundary, selecting result two gives a preceding byte destination of 16,384 and a halfword location of 8,248. The consumer receives 8,192 first and the halfword read second.

For saved address 256, the scaled byte destination is 65,536, the later halfword location is 32,800, and the direct word destination is 256. Accessibility of one does not establish accessibility of either other location.

### Paired mutation and unresolved arithmetic

An established retained-product pattern of 2,684,354,560 supplies three-bit datum five. This assumes the pattern; it does not assert that preceding signed arithmetic validly produces it.

With late selecting result seven, byte five is written at address 56 before word 28 is written through the saved address. The following interaction receives **28, 28, 28**.

Overlap does not eliminate either write. The resulting overlapping bytes additionally depend on byte order. No acceptance example assigns a result to the unresolved 32-position shift.

## Uncertainties

The following remain **documented_only**:

- Application purpose and completeness of the represented interface.
- Actual calling convention, hidden inputs, and secondary-output production or preservation.
- External dependency identities, errors, termination, and side effects.
- Signed-overflow behavior, negative-operand shifts, address conversion, and the out-of-width shift.
- Whether wide results are single values or paired outputs.
- Storage mappings, extents, lifetimes, ownership, and overlap.
- Alignment legality, byte order, atomicity, synchronization, and halfword argument extension.
- Global storage width, meaning, and preservation after assignment.
- Actual handling of the terminal mechanism.

**[INFERENCE]:** Repetition may reflect mechanically generated behavior. It does not establish a bulk-operation purpose or authorize collapsing interactions.

Annotations concerning omitted material do not establish the behavior or reachability of material absent from the supplied representation.

## Evidence

This contract uses static examination of the supplied representation for F-b9d3f895d4bd. Prior prose, reviewer feedback, annotations, and apparent symbol meanings are not independent behavioral evidence.

Established findings concern access widths, data provenance, relative ordering, retained-result relationships, argument positions, the byte-write inventory, two halfword reads, the global assignment, the direct word mutation, subsequent activity, and the represented non-returning terminal path.

Dependency bodies and platform contracts were not supplied. No runtime tests have been performed. Acceptance examples are conditional deductions, not observed execution results.
