<!-- Trusted provenance: F-6fd1b66bda02; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fd1b66bda02

## Purpose

F-6fd1b66bda02 accepts three 32-bit input words, performs an extensive ordered sequence of indirect interactions, mutates memory through computed and fixed addresses, and invokes a final unresolved target with four argument words.

An application-level purpose is not established. The visible behavior does not justify describing the function as a formatter, allocator, initializer, serializer, or device controller. Address-linked writes and punctuation-like constants are observable details, not evidence of a particular application model.

This contract distinguishes:

- **Established static evidence:** widths, data dependencies, accesses, and sequencing explicitly represented in the supplied material.
- **Inferred intent:** possible interpretations that do not impose acceptance requirements.
- **documented_only:** unresolved behavior for which no executable requirement can be established.

Static evidence describes the supplied representation; it does not independently establish executable accuracy. No runtime tests have been performed.

## Interface

### Explicit inputs

| Position | Width | Established use |
|---|---:|---|
| First | 32 bits | No explicit value use is represented. |
| Second | 32 bits | No explicit value use is represented. |
| Third | Unsigned 32 bits | Supplies two early argument constructions dependent on its low byte. |

Under 32-bit truncating shift semantics, the third input contributes:

- Its low byte placed in bits 24–31 of a word.
- Its low byte multiplied by 32, ranging from zero through 8,160.

These occur in separate interactions. The first construction accompanies an explicit zero argument; the second accompanies a shifted dependency result.

The first two inputs remain part of the interface. Incompletely recovered calling conventions leave their possible visibility to indirect targets **documented_only**. Likewise, equality of the third input’s low byte establishes equality only of the two explicit constructions, not equality of complete executions.

### Implicit inputs

Execution also depends on incoming control state, indirect-target behavior, result-register contents, existing memory, fixed-address storage, and the final unresolved target.

Represented dependency results include 32-bit words and 64-bit quantities subsequently separated into numerical low and high 32-bit words. Additional register-derived words have incompletely specified producers. A represented 64-bit result does not establish a dependency’s public return type or memory byte order.

### Outputs and completion

There is no explicit return value. Outputs consist of dependency interactions, memory effects, and the final four-word interaction.

The final argument positions are:

| Position | Established value or provenance |
|---|---|
| First | 0x72 |
| Second | A late word read at offset 0x64 from a base obtained through a computed offset-0x04 word read |
| Third | An earlier saved word read at offset 0x14 from the final mutation root |
| Fourth | A later word read at offset 0x74 from the same base used for the second argument |

The read times differ substantially. These arguments are not a simultaneous snapshot.

A return is represented after the final interaction. Normal completion requires that target to return; its termination, exception, and continuation behavior is **documented_only**.

### Preconditions and errors

No explicit validation, bounds checking, alignment checking, or error-result protocol is visible.

A completing execution requires every reached access and control transfer to be supported by the execution environment. Computed addresses can be small, and fixed accesses include very low and unaligned addresses. Conventional process-memory assumptions are not established.

No caller-owned buffer, allocation extent, or ownership-transfer precondition is identified.

## Behavior

### Indirect interaction phase

Before the direct-memory phase, the function performs a long, ordered sequence of indirect interactions. Some have no represented explicit arguments; others receive one through four words. Explicit zeros, duplicated arguments, shifted results, and saved results reused after intervening interactions are all represented.

Interactions remain potentially observable even when their results are unused. Absence of represented arguments does not establish absence of inputs, state changes, or failures.

The first target depends on incoming execution state. Subsequent targets are constructed from represented control-location values and a fixed displacement. Their identities and contracts are unresolved. Similar argument patterns do not establish target equivalence.

No explicit loop or conditional branch is visible in this phase. Dependencies can nevertheless branch, block, fail, mutate memory, or fail to return. They must not be assumed to preserve globals, memory ownership, or previously observed storage.

### Width and arithmetic requirements

The data path distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and represented 64-bit results.

Narrow stores retain the low bits appropriate to their width. Byte loads widened into words are zero-extended where explicitly represented. Unsigned right shifts are logical. Wide-result extraction occurs at the represented width before conversion to a word.

Recoverable field bounds include:

| Construction | Numerical range |
|---|---:|
| Low 20 bits, logically shifted right by 10 | 0–1,023 |
| Low 24 bits, logically shifted right by 20 | 0–15 |
| Low 24 bits, logically shifted right by 19 | 0–31 |
| Low 24 bits, logically shifted right by 14 | 0–1,023 |
| Low 22 bits, shifted left by eight | 0–0x3FFFFF00 |
| Low 19 bits, shifted left by two | 0–0x1FFFFC |

Unlimited-precision substitution is not acceptable. Under a truncating 32-bit interpretation, a left shift by 25 followed by another by 11 produces zero. A word shifted left by eight and then logically right by eight retains its original low 24 bits.

Some scaling, subtraction, and left shifts use represented signed words. Their exact overflow and address-wrap semantics are **documented_only**. Modular arithmetic examples in this contract are conditional, not claims of portable signed-arithmetic behavior.

A late byte value is derived from a signed right shift whose count is the low eight bits of a computed word. Negative-operand behavior and counts of 32 or greater remain unresolved.

### Initial direct-memory effects

The direct phase begins using the final represented wide result and previously saved dependency-derived values.

A word store through the high result word’s offset 0x14 precedes a byte store at another computed base’s offset 0x09. A word store through a low-result-derived scaled base at offset 0x54 follows. The offset-0x09 byte is then overwritten with another value.

Another computed region has its offset-0x14 word cleared before its offset-0x05 byte is read. A second region’s offset-0x14 word is cleared; the saved byte is widened into a word stored at the first region’s offset 0x34. The first cleared word is then read. The second region’s word is cleared again, followed by two byte writes at its offset 0x15.

These are access-order requirements, not guarantees that cleared fields still contain zero. Computed destinations can alias or overlap.

### Early fixed-address interactions

After those initial computed writes, the function reads a byte at fixed address 0x05 and zero-extends it for use as an address base. That read precedes the byte write at fixed address 0x01, whose value is the low byte of a computed doubled base.

The saved value from 0x05 subsequently determines an offset-0x15 byte destination and an offset-0x54 word location. The latter location is read before being overwritten. Its saved word becomes an important source for later scaled bases and cross-linked writes.

Later, after additional computed reads and stores:

- A word at fixed address 0x54 receives the saved word shifted left by one.
- Computed offset-0x14 word writes and repeated offset-0x15 byte writes follow.
- Fixed address 0x15 then receives the low byte of the computed doubled base.

The byte at 0x05 is not reread for these uses. Its saved value and later memory contents are distinct concepts. No non-aliasing assumption permits moving the read after the 0x01 write or moving the fixed stores across intervening accesses.

### Address-linked mutation and layout

Subsequent operations load words as address bases and publish computed addresses into other locations. Some resemble self-links or cross-links. Object identity, list membership, reference counting, and ownership are not established.

The access footprint includes:

| Offset from a computed base | Represented access |
|---|---|
| 0x01 | Byte write |
| 0x04 | Word read; some regions also supply byte reads here |
| 0x05 | Byte read |
| 0x08, 0x09, 0x0D, 0x11, 0x15, 0x19 | Byte writes |
| 0x10 | Byte and word writes |
| 0x14 | Word reads and writes |
| 0x20 | Repeated word writes |
| 0x24, 0x34, 0x44, 0x54 | Word writes; some are also read |
| 0x40, 0x64, 0x74 | Word reads; 0x74 is also written |

Scaled addresses without field offsets receive byte, halfword, or word stores.

This is not one structure declaration. In particular, an offset-0x15 byte overlaps the four-byte range beginning at offset 0x14. Repeated accesses cannot be collapsed merely because a final-state calculation appears equivalent.

### Constants and repeated publication

A later region writes 0x3A as a byte through a fourfold-scaled address, as a halfword through an eightfold-scaled address, and as a word at offset 0x20 from the fourfold-scaled address.

A separate computed word destination at offset 0x20 receives, in order:

- 0x20;
- 0x2F;
- a zero-extended byte read at offset 0x04 from the same computed base;
- 0x20 again.

After these writes, a word is loaded from fixed storage at 0x087FBBBC. **This base load precedes the word store of 0x20 at fixed address 0x60.** The loaded base is saved for subsequent accesses. It is not obtained by rereading storage after the 0x60 write. Unknown storage relationships or address mapping do not justify reversing these events.

The following publication region repeatedly associates a scaled destination, the loaded base, and another computed workspace. Each represented group preserves this order:

- An offset-0x54 word publication.
- An offset-0x19 byte write through the loaded base.
- An offset-0x11 byte write through the workspace.
- An offset-0x34 word publication through the loaded base.

The first group uses a different scaled destination association from the subsequent groups. The byte values are low bytes of address-derived words, not established Boolean flags. Every represented repetition remains part of the observable access sequence; an implementation that performs only the final group is insufficient where stores are observable.

### Final-region terminology

For the remaining behavior:

- **Mutation root** means the doubled base obtained during the preceding address-linked accesses.
- **Workspace** means the computed base used for the repeated offset-0x11 byte writes.
- **Publication base** means the word saved from fixed address 0x087FBBBC.
- **Fourfold-linked region** means the fourfold-scaled linked base used for the preceding 0x3A byte store.

These are descriptive labels, not assertions of object boundaries or ownership.

### Final fixed storage and saved reads

After the publication region, a word at fixed address 0xA9 is read and saved. A word is then read from the fourfold-linked region’s offset 0x74. Both reads precede the byte write at fixed address 0x3A. That byte is an address-derived value formed by eightfold scaling and addition of 0x30, narrowed to eight bits.

The mutation root’s offset-0x14 word is then read and saved for the final third argument. A word from workspace offset 0x04 is also saved. Several computed stores follow, including byte writes through the mutation root and workspace and a self-address word publication at mutation-root offset 0x54.

Only afterward does fixed address 0x67 receive the low byte of the previously saved 0xA9 word.

A computed word read at mutation-root offset 0x44 occurs next, before the read of fixed byte 0x67 used for the write to fixed byte 0x21. Thus the 0x21 write uses the actual later 0x67 read, not an abstract copy of the original 0xA9 low byte.

Further computed writes follow. These include a workspace offset-0x10 byte write, two successive publications at publication-base offset 0x54, and the shift-dependent byte write through mutation-root offset 0x01.

A workspace offset-0x40 word read supplies a later workspace offset-0x10 word store, after the earlier byte store there. The saved workspace offset-0x04 word is published at publication-base offset 0x74.

Two additional workspace offset-0x09 byte stores explicitly reread fixed address 0x67 at their respective positions. Between those reads are computed writes, including a mutation-root offset-0x09 byte write and publication of the saved 0xA9 word at workspace offset 0x44. A workspace offset-0x54 word read also precedes the second of these additional 0x67 reads.

These global reads cannot be replaced with the originally saved byte: intervening computed writes may alias or overlap fixed storage.

### Final argument provenance and transfer ordering

The word read from workspace offset 0x54 becomes a destination base for subsequent stores. Its offset 0x0D receives the low byte of the earlier fourfold-linked-region offset-0x74 read. The mutation root’s offset 0x54 receives the saved root offset-0x14 word. That saved word remains the final third argument even if either location is subsequently changed.

A separate workspace offset-0x14 word is read later. Its low byte supplies an upcoming destination offset-0x11 store; it is **not** the final third argument.

After further computed byte and word writes, a word read at mutation-root offset 0x04 supplies another base. A halfword containing 0x65 is stored at twice that base before its offset-0x64 word is read. This offset-0x64 read supplies the final second argument.

The destination offset-0x11 byte store then uses the saved workspace offset-0x14 low byte. The destination’s offset-0x54 word receives decimal 100. After that store, the base obtained through mutation-root offset 0x04 supplies an offset-0x74 word read: this is the final fourth argument.

Only afterward is another word read, this time at mutation-root offset 0x64. That value supplies a further destination base. The remaining writes occur before transfer:

- A mutation-root offset-0x09 byte receives the low byte of the base obtained through offset 0x04.
- The further destination’s offset-0x54 word receives the saved second argument.
- A scaled word destination at twice the saved second argument receives the value read from mutation-root offset 0x64.

The final target then receives 0x72, the saved second argument, the earlier saved root offset-0x14 word, and the saved fourth argument. Later aliasing writes do not retroactively change saved argument words.

## Side effects

Observable effects include indirect interactions, mixed-width memory reads and writes, fixed-address accesses, repeated publications, address-derived byte values, and the final control transfer.

There is no visible rollback or transaction boundary. Failure after earlier stores does not establish restoration of those stores or production of an error code.

No explicit allocation or release operation is identifiable. Indirect dependencies may allocate, retain, transfer, or release memory; none of these behaviors can be assumed absent.

Atomicity, synchronization, memory barriers, thread safety, reentrancy, and global preservation are **documented_only**. Represented access order must be preserved without claiming an independently established hardware memory-order model.

## Dependencies

F-6fd1b66bda02 is the only supplied opaque function ID with an established mapping. The indirect dependencies and final target have no supplied opaque IDs or resolved contracts. Their identities are **documented_only**; additional IDs are not invented.

Unknown dependency behavior does not erase visible caller-side facts. Argument widths, constructions, saved-read provenance, and ordering remain established where explicitly represented.

No unresolved dependency is assumed to preserve globals, use only represented arguments, return normally, or own or release passed values.

## Acceptance examples

These examples are conditional requirements, not executed tests.

### Low-byte boundaries

With truncating 32-bit shifts, third inputs 0x000000AB and 0x123456AB produce the same two explicit input-derived words: 0xAB000000 and 0x1560.

A low byte of zero produces two zeros. A low byte of 0xFF produces 0xFF000000 and 8,160. Equal complete behavior is not required.

### Fixed-base read before write

Suppose the load from 0x087FBBBC observes a particular base and the later 0x60 word store affects storage visible through that location. Subsequent publications still use the saved base. Reloading after the store is not equivalent.

### Fresh fixed-byte observations

Suppose the initial write to 0x67 stores 0x12, but an intervening computed store changes that byte to 0x34 before a later explicit read. The later consumer receives 0x34. Substituting the saved 0xA9 low byte is unacceptable.

The computed root offset-0x44 read must remain before the 0x67 read used for the 0x21 write.

### Saved third argument

Suppose the early mutation-root offset-0x14 read returns 0x11223344, and later writes change that field to 0x55667788. The final third argument remains 0x11223344.

### Late argument timing

If the scaled halfword store of 0x65 overlaps the storage subsequently read for the second argument, that read observes the post-store state. The fourth argument is read after the decimal-100 word store and before the later mutation-root offset-0x64 read and final stores.

### Mixed-width overlap

If an offset-0x15 byte store overlaps a cleared offset-0x14 word, only its addressed byte is written by that narrow access. The resulting numerical word depends on byte order. The repeated stores remain individually represented.

### Partial failure and continuation

If a later access fails, no rollback is established. If execution reaches a final target that does not return, normal return from F-6fd1b66bda02 is not required.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and success criteria.
- Accuracy of the supplied representation relative to executable behavior.
- Dependency identities, calling conventions, and additional result-register provenance.
- Indirect visibility of the first two explicit inputs.
- Byte order, alignment support, address mapping, and low-address validity.
- Signed overflow, address wrap, and exceptional shift behavior.
- Allocation extents, object boundaries, aliasing, ownership, and lifetime.
- Significance of repeated stores and final-target continuation behavior.

Possible textual or linked-object intent is inference only. Neither unusual fixed addresses nor repetition justifies deleting represented effects or declaring them unreachable.

## Evidence

This contract is based solely on static inspection of the supplied representation, independently of the prior draft’s assertions.

The represented accesses establish the fixed-base load before the 0x60 store; the ordered 0x05, 0x01, 0x54, and 0x15 interactions; the saved 0xA9 word and subsequent fresh 0x67 reads; and the distinct read times and sources of the final arguments.

No implementation access, dependency documentation, executable trace, or runtime test evidence was available. All acceptance examples are conditional and unexecuted.
