<!-- Trusted provenance: F-d7c0ba6f8cfe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7c0ba6f8cfe

## Purpose

F-d7c0ba6f8cfe performs an extensive dependency-driven computation, applies an ordered sequence of mixed-width memory updates, and invokes an unresolved terminal target.

The supplied static representation establishes no application-level purpose. It does not justify interpreting the function as text formatting, object initialization, container manipulation, or a known protocol. Character-like constants and recurring offsets establish numeric operations, not semantic field types.

This contract distinguishes:

- **Established static evidence:** widths, transformations, accesses, and ordering directly supported by the supplied representation.
- **[INFERENCE]:** possible intent, not a behavioral requirement.
- **documented_only:** unresolved behavior or an explicitly unexpanded part of the behavioral boundary.

The contract describes represented behavior without asserting that every operation has established executable semantics. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit consumption in the visible body. |
| Second argument | 32 bits | No explicit consumption in the visible body. |
| Third argument | Signed 32 bits | Supplies left-shifted values to early dependency invocations. |

The first invocation receives two equal values formed by shifting the third argument left by 10 bits. A later invocation receives that argument shifted left by 26 bits. No visible range check precedes either transformation.

Absence of explicit use does not prove that the first two arguments are behaviorally irrelevant. The indirect-call interfaces are unresolved; their targets may consume machine state not expressed in the represented argument lists.

### Implicit inputs

The function also depends on incoming control state, indirect destinations, primary and secondary result channels, global storage, and memory reached through computed or loaded addresses.

Some addresses derive from individual bytes, scaled integers, or masked results. There is no visible requirement that these values originate from conventional allocated objects.

### Outputs and completion

There is no explicit result value. Observable outputs include dependency invocations, memory reads and writes, intermediate memory states, and the terminal invocation.

The terminal target receives four represented word arguments:

1. 0x72.
2. A word loaded from a late +0x64 location.
3. A word saved from a computed +0x14 location near the beginning of final propagation.
4. A word subsequently loaded from a late +0x74 location.

A return follows the terminal invocation in the representation. Whether the actual transfer returns under an ordinary call convention is **documented_only**.

### Preconditions

Successful completion requires executable destinations, compatible calling conventions, valid accesses at every reached width, and defined arithmetic and shift behavior in the execution environment.

The function visibly provides no address validation, allocation-bound checks, null handling, dependency-status checks, or alternate recovery path. These are execution prerequisites, not conditions the function enforces.

## Behavior

### Dependency-driven computation

Before the first explicit data-memory store, the function performs a long ordered sequence of indirect invocations. Represented calls have zero to four explicit word arguments. Their results are used through 32-bit values, paired 32-bit components, and incompletely specified secondary result channels.

The computation includes duplicated arguments, explicit zeros, retained earlier results, left shifts, unsigned right shifts, and masks selecting low-order fields. Some results survive many intervening invocations before reuse.

These invocations are observable dependency boundaries, not a pure expression. A call remains significant even when its result is unused or its arguments reduce to zero. No assumption of purity, successful return, global preservation, or memory preservation is permitted.

The visible body has no retry loop or branch testing dependency success. Dependencies may have their own control flow or may prevent later operations from being reached.

### Widths and arithmetic

The represented access widths are:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word: 32 bits.
- Selected call-result aggregate: 64 bits, separated into low and high 32-bit components.

Byte stores retain the low eight bits. Unsigned byte loads widened to words produce values from 0 through 255, without sign extension.

Unsigned 32-bit left shifts discard bits beyond bit 31. A subsequent right shift cannot recover discarded bits. Signed left shifts, signed multiplication overflow, and signed address arithmetic are not unconditionally assigned modular semantics by this contract.

Masks include 0xFFFFFF, 0x3FFFFF, 0x1FFFFF, 0xFFFFF, 0x7FFFF, 0x3FFFF, 0x1FFFF, 0x7FFF, 0x3FFF, 0x1FFF, 0xFFF, and 0x7FF. These retain respectively 24, 22, 21, 20, 19, 18, 17, 15, 14, 13, 12, and 11 low bits.

Consequently:

- A low-20-bit selection shifted right by 10 contains at most 10 bits.
- A low-24-bit selection shifted right by 19 contains at most five bits.
- A 64-bit unsigned result shifted right by 34, 36, or 44 uses its high component shifted right by 2, 4, or 12, respectively.

Near the end, a signed word is right-shifted by a count taken from the low byte of a computed address. The count can exceed 31. Its result is **documented_only** until the target’s signed-shift and excessive-count semantics are established. Reducing the count modulo 32 is not justified.

### Layout and aliasing boundary

Recurring byte offsets include +0x01, +0x05, +0x08, +0x09, +0x0D, +0x10, +0x11, +0x15, and +0x19. Recurring word offsets include +0x04, +0x14, +0x20, +0x24, +0x34, +0x40, +0x44, +0x54, +0x64, and +0x74.

These are access locations, not a recovered structure definition. Word and byte fields can overlap: a word at +0x14 includes the byte at +0x15; a word at +0x10 includes the byte at +0x11.

No object sizes, disjointness guarantees, allocation provenance, or ownership relationships are established. Unless an acceptance scenario supplies stronger memory assumptions, later loads must observe memory at their own represented read points.

### Initial memory updates

The opening stores publish a retained word relative to the final dependency result’s high component, write a byte at a retained address plus nine, write a word at an address derived from the result’s low component multiplied by 256 plus 0x54, and overwrite the earlier plus-nine byte with another value.

The following clear-and-publication boundary has a specific order:

1. Clear the first computed base’s +0x14 word.
2. Read and save its unsigned +0x05 byte.
3. Clear the second computed base’s +0x14 word.
4. Publish the saved byte, zero-extended to a word, at the first base’s +0x34.
5. Read the first base’s +0x14 word.

The byte read is therefore between the two clears, not after both. Publishing the saved byte is also different from reloading it after the second clear. Any equivalence argument must establish the relevant noninterference rather than presume it.

A further clear of the second base’s +0x14 word precedes two identical byte stores at its +0x15 location. Both stores remain represented events.

An unsigned byte is then read from absolute address 0x05 and retained as an address base. A byte is written at absolute address 0x01. Additional stores update the byte-derived base and the first computed base, including a temporary +0x11 value followed by zero.

A word is written at +0x54 relative to the earlier +0x14 readback. Under ordinary memory, without intervening aliasing or external modification of that cleared word, the readback is zero and this destination is absolute address 0x54. This consequence does not establish that low-address storage is invalid.

### Linked and scaled updates

The function reads the byte-derived base’s +0x54 word before replacing it. The former contents drive later address calculations, including scaling by two, four, and eight.

This phase includes self-valued +0x24 stores, repeated byte stores, successive +0x14 publications, +0x34 and +0x54 updates, and explicit writes to absolute addresses 0x54 and 0x15.

A loaded +0x74 word becomes the base of a +0x54 write. Another pointer-following operation contains separate reads of a +0x74 location. Their ordering relative to each other inside that expression is not established here, but caching them as one read is not justified.

Other traversal addresses are obtained by widening a +0x05 byte, adding 0x74, and reading a word there. Such a byte directly contributes to the address; no hidden table base is established.

Subsequent updates propagate low bytes into +0x0D and +0x09 locations, temporarily publish self values through +0x14, follow +0x64 words, and overwrite +0x44 fields. Repeated +0x05 reads and later byte-derived +0x74 traversals remain distinct. Earlier stores can affect later reads where memory aliases.

### Constant-width stores and publication region

A word obtained through a byte-derived +0x74 traversal supplies scaled destinations for three different stores:

| Destination relationship | Width | Value |
|---|---:|---:|
| Four times the loaded word | 8 bits | 0x3A |
| Eight times the loaded word | 16 bits | 0x003A |
| Four times the loaded word, plus 0x20 | 32 bits | 0x0000003A |

These are not interchangeable character writes.

For the following discussion, the **adjusted base** is four times the result of subtracting 0x96 from an earlier fourfold-scaled value. Its +0x20 word receives, in order, 0x20, 0x2F, a widened unsigned byte read from adjusted-base +0x04, and 0x20 again. The byte read occurs after the 0x2F store and before its own publication.

Immediately after the last +0x20 write, the function reads a separate global word and retains it as the **global-loaded base**. Only after that read does it write the word 0x20 at absolute address 0x60. The global read must not be moved after the 0x60 store or replaced with an entry-time snapshot.

A long, finite repeated publication region follows. Each group contains, in order:

- A word publication through a scaled base’s +0x54.
- A byte at global-loaded-base +0x19 containing the adjusted base’s low byte.
- A byte at adjusted-base +0x11 containing the global-loaded base’s low byte.
- A scaled-address word at global-loaded-base +0x34.

The first group uses the pre-adjustment eightfold scale for its word destinations and publication. Subsequent groups use the adjusted eightfold scale. No explicit dependency call occurs within this region.

The exact group count is **documented_only** in this contract: it is not enumerated here, and no particular trace cardinality is asserted. The region must not be treated as one group or as an execution-dependent retry.

[INFERENCE]: The pattern resembles linkage or publication of related state. It establishes neither a container type nor synchronization or ownership transfer.

### Final propagation: saved values and low-address reads

The **doubled traversal base** is twice the word whose fourfold scale supplied the 0x3A destinations and repeated +0x54 publication value.

Final propagation opens with these ordered boundaries:

1. Read the global word at absolute address 0xA9.
2. Read the +0x74 word relative to the fourfold traversal value.
3. Write the computed byte at absolute address 0x3A.
4. Save the doubled traversal base’s +0x14 word.
5. Save the adjusted base’s +0x04 word.

The 0x3A byte is the low-byte result of multiplying the adjusted predecessor by eight and adding 0x30, subject to the arithmetic qualifications above. The +0x14 and +0x04 saves occur after this write. If their storage overlaps that write, the saved values reflect it.

Next, low address-derived bytes are propagated through +0x0D and +0x09 fields, adjusted-base +0x08 receives 0x6C, and doubled-traversal-base +0x54 temporarily receives its own base.

The function writes the low byte of the saved 0xA9 word to absolute address 0x67. It then reads doubled-traversal-base +0x44 as a word. Only afterward does it read the byte at 0x67 and write that byte to absolute address 0x21. The +0x44 read therefore precedes the 0x21 write; it cannot be moved after that copy without a noninterference justification.

The loaded +0x44 word supplies a byte at adjusted-base +0x10 and a temporary word publication at global-loaded-base +0x54. Its signed right-shift result supplies doubled-traversal-base +0x01. The temporary publication is then overwritten with the adjusted base.

A word read from adjusted-base +0x40 is retained before the saved +0x04 word is published at global-loaded-base +0x74. The retained +0x40 word is then written at adjusted-base +0x10, followed by a word update at global-loaded-base +0x44.

There are two additional, separate reads of absolute-address byte 0x67 for adjusted-base +0x09 stores:

- **First later read:** after the global-loaded-base +0x44 update, read 0x67 and store that byte at adjusted-base +0x09.
- Between the two reads, write the adjusted base’s low byte at doubled-traversal-base +0x09, write the saved 0xA9 word at adjusted-base +0x44, and load adjusted-base +0x54 as a new base.
- **Second later read:** after that +0x54 load, read 0x67 again and store that byte at adjusted-base +0x09.

Neither later store is specified to use a cached low byte from the saved 0xA9 word, the earlier 0x67 read, or the other later read. Intervening computed stores may overlap or otherwise alias 0x67.

### Final propagation: completion boundary

The saved traversal +0x74 word supplies low bytes for the new base’s +0x0D and doubled-traversal-base +0x11. The previously saved doubled-traversal-base +0x14 word is restored through its +0x54.

A separate adjusted-base +0x14 word is then saved. The adjusted base’s +0x09 and +0x44 locations are subsequently overwritten with the low byte and full word of the new base.

The function reads doubled-traversal-base +0x04, writes halfword 0x0065 at twice that loaded value, and follows its +0x64 word. The saved adjusted-base +0x14 word supplies the new base’s +0x11 byte; the new base’s +0x54 word receives decimal 100.

A +0x74 word relative to the loaded +0x04 value is saved for the terminal invocation. A later doubled-traversal-base +0x64 read supplies another base. Before the terminal invocation, the function updates doubled-traversal-base +0x09, writes the earlier +0x64 word through the later base’s +0x54, and writes the later base as a word at twice the earlier +0x64 value.

Saved values remain tied to their read points. Replacing them with later reloads is not generally equivalent.

## Side effects

The function is not established as pure, atomic, reentrant, or thread-safe.

Direct effects include mixed-width stores, low-address accesses, pointer-following reads, repeated writes, temporary publications, overlapping fields, and address-derived byte truncation. Final memory contents alone do not capture all represented behavior.

Dependency effects are unrestricted by the available evidence. Dependencies may modify globals or reachable storage, allocate or release resources, fail, or redirect control. No preservation or resource-ownership convention is assumed.

No explicit allocation or release operation is identifiable in the visible body. No caller cleanup obligation, borrowing duration, or ownership transfer can be assigned.

There is no visible rollback, status result, exception translation, or recovery action. Failure after earlier effects does not imply restoration.

## Dependencies

The following IDs are contract-local opaque dependency labels, not recovered target identities.

| Opaque ID | Established boundary | Unresolved behavior |
|---|---|---|
| F-000000000001 | Ordered indirect invocations with zero to four represented word arguments; word and paired-component results | Target identities, complete arguments, secondary channels, state changes, ownership, failures, termination |
| F-000000000002 | Terminal invocation with 0x72 and three saved or loaded words | Transfer convention, effects, ownership, failure and return behavior |

F-000000000001 denotes a dependency set and may contain many distinct functions. A no-argument presentation does not establish a target with no machine-state inputs. A represented 64-bit aggregate does not establish an application-level scalar return type.

## Acceptance examples

These examples are conditional static criteria, not executed tests.

1. **Argument transformation.** Under 32-bit modular shift semantics, a third argument of one supplies two 0x00000400 values to the first invocation and supplies 0x04000000 to the later direct transformation. Dependency results remain unconstrained.

2. **Discarded bits.** Under unsigned 32-bit semantics, shifting 0x01000000 left by eight yields zero. Later right shifts cannot recover its discarded high bit.

3. **Initial saved byte.** If the first base’s +0x05 read yields 0xFE, the later +0x34 publication is 0x000000FE. It uses that saved read even if the intervening second-base clear changes storage visible through the byte location.

4. **Overlapping fields.** A +0x14 word clear followed by a +0x15 byte store preserves both events. The resulting word depends on byte order and the byte value.

5. **Global-base capture.** If the 0x60 store affects storage supplying the global-loaded base, the retained base still comes from the earlier read, after the final adjusted-base +0x20 write and before the 0x60 store.

6. **0x3A overlap.** If the 0x3A byte write affects a subsequent saved +0x14 or +0x04 word, that save observes the changed storage. Reading both words before the byte write is not acceptable.

7. **0x21 overlap.** If the 0x21 byte write affects storage covered by the computed +0x44 word read, the retained word is nevertheless the value read before the 0x21 write.

8. **Fresh 0x67 reads.** If an intervening computed store changes 0x67, the following adjusted-base +0x09 store uses the byte read afterward. The second later read remains after the computed +0x54 load, even if an ordinary-memory scenario would make the two reads equal.

9. **Intermediate publication.** A trace-sensitive scenario preserves the adjusted-base +0x20 values 0x20, 0x2F, widened loaded byte, and 0x20 in order. Matching only the last value is insufficient.

10. **Failure and return.** An invalid reached access has no promised rollback. If all operations complete and F-000000000002 returns normally under the actual convention, the represented body then returns without an explicit result.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid input domain, and object model.
- Actual dependency identities, full interfaces, and secondary result channels.
- Signed overflow, negative left shifts, excessive shift counts, and executable interpretation of represented arithmetic.
- Byte order, alignment, memory mapping, low-address validity, and physical aliasing.
- Allocation provenance, ownership, concurrent access, device behavior, and fault observability.
- Evaluation order between reads contained within a single represented expression.
- Exact repeated-publication cardinality, unexpanded in this contract.
- Dependency failure propagation and terminal control-transfer semantics.

These limits prevent treating this document as a complete executable replacement specification.

## Evidence

The supplied static body supports the explicit widths, dependency-driven transformations, mixed-width accesses, ordered overwrites, saved-value reuse, and terminal argument origins described here.

In particular, it supports the byte read between the initial clears; the global-base read before the 0x60 store; the 0x3A write before the final +0x14 and +0x04 saves; the +0x44 read before the 0x67-to-0x21 copy; and both later, separate 0x67 reads, including the second after the computed +0x54 load.

Comments, prior prose, and reviewer assertions were not treated as independent proof of intent. No dependency bodies, runtime traces, target ABI, or memory-map specification were supplied. No runtime tests have been performed, and no provenance hash is asserted here.
