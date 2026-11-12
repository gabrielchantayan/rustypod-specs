<!-- Trusted provenance: F-afd60aafb68b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afd60aafb68b

## Purpose

F-afd60aafb68b transforms address-bearing records along one of two entry routes. One route prepares a record and then chooses interrupt processing or selected-record processing. The other initializes a construction record and chooses publication or an unresolved continuation.

Its contract is defined by observable memory effects and control transfers, not by an established application-level success condition. Publication is a group of writes, not evidence of synchronization or ownership transfer. Completion installs fields but does not establish successful termination.

Effects are not transactional. A fault, nonreturning dependency, or exceptional operation can prevent later effects without undoing earlier ones.

The descriptions below distinguish established static behavior from derived consequences and unresolved environmental behavior. No runtime tests have been performed.

## Interface

### Explicit inputs and inherited context

The represented interface has four 32-bit argument positions. No explicit use of the first two is established. The third supplies the **initial primary address**; the fourth supplies the **initial secondary address**.

Additional inputs come from inherited machine context or externally stored values:

- An entry-selection byte and a link-associated value.
- A construction-state word and a preliminary-write displacement.
- A coprocessor operand.
- A stack-associated word at displacement 0x264.
- Stack-associated storage beginning at displacement 0x29C.
- A shared displacement used to sample a selected record and choose its continuation.
- Embedded replacement inputs, a replacement descriptor, a construction tag, and a completion argument.
- Values associated with numeric locations 0x69 and 0x76.
- An unresolved construction-continuation destination.

These are not established additional formal parameters. Their initialization, lifetime, and observation times where not specified below are **documented_only**.

### Representation

Addresses and words are 32 bits, halfwords are 16 bits, and bytes are 8 bits. Pair-valued results have distinct lower and upper 32-bit components.

Address arithmetic, doubling, subtraction, and ordinary addition retain the low 32 bits. Narrow writes retain only the destination width. Adjusting a pair’s lower component does not carry into its upper component.

Signed interpretation matters in three places:

- Classifying the initial primary address after shifting it left fourteen bits and retaining 32 bits.
- Detecting signed overflow when adding 0x32 to an already-truncated double of the initial secondary address.
- Arithmetically shifting a publication pair’s lower component right five bits.

Accessible storage of the required width is a precondition for every reached access. No bounds validation, alignment validation, reservation, or recovery from invalid access is established. Some specified accesses are unaligned.

### Outputs and termination

There is no established ordinary return-value contract or error-result convention.

The only represented ordinary return belongs to the non-overflow construction continuation, and occurs only if its unresolved final dependency returns. Other routes reach exceptional operations. Their resumability is not established.

## Behavior

### Entry decision

F-0001 is the first represented interaction. Its destination is the inherited link-associated value plus 0xFAA, with 32-bit wrapping. Its result supplies a lookup base used only by construction.

The entry-selection test follows this interaction: zero selects preparation; nonzero selects construction. Whether the tested byte was captured before F-0001 is unresolved. Consequently, the contract does not assume either that F-0001 preserves its underlying state or that its effects necessarily change the value tested.

All subsequent behavior is conditional on earlier accesses succeeding and earlier dependencies allowing continuation.

### Preparation and selection

Preparation uses three address-derived values:

- The **prepared address** is twice the initial primary address, truncated to 32 bits.
- The **mixed address** is the bitwise exclusive-or of the initial primary address and prepared address.
- The **discriminator** is the initial primary address shifted left fourteen bits, truncated to 32 bits, and interpreted as signed.

F-0002 consumes the halfword stored at twice the mixed address plus 0x26, together with 0x20. It receives the stored payload, not the address of that payload. Extension of the halfword to argument width is unresolved.

The upper component of its result becomes the **preparation word**. The lower component minus five, with wrapping, becomes the **reduced word**.

The prepared record has the following established initialization fields:

| Offset | Width | Installed value |
|---|---|---|
| 0x00 | Word | Prepared address |
| 0x04 | Word | Prepared address |
| 0x08 | Word | Prepared address |
| 0x0C | Word | Reduced word |

Initialization is not the first memory effect after F-0002. A word containing the preparation word is first written at mixed address plus preparation word. The word at prepared-address offset 0x18 is then retained as an interrupt destination. A byte containing the prepared address’s low eight bits is written at prepared address plus preparation word. Only then are the four initialization fields written, in ascending offset order.

Selection occurs after those writes. The selected address is read as a word at prepared address plus 0x50 plus reduced word. An unsigned selector byte is subsequently read at twice the sum of the prepared address and 0x50. Finally, the selected address’s low sixteen bits are written at numeric address selector plus 0x3E.

The last destination is selector-relative, not prepared-record-relative. Its starting address ranges from 0x3E through 0x13D.

These reads do not constitute an entry snapshot. Earlier writes can change later observations through aliasing. In particular, the first word write can affect the retained interrupt destination, and record initialization can affect either selection read.

### Nonnegative discriminator: interrupt processing

A nonnegative discriminator selects interrupt processing. A strictly positive discriminator additionally causes a coprocessor interaction using selectors 9 and 5, the selected address, the reduced word, and the inherited coprocessor operand. A zero discriminator omits that interaction.

F-0003 normally receives the unsigned selector. It instead receives selector minus 0x170, with 32-bit wrapping, when both of these conditions hold:

- The discriminator is strictly positive.
- The prepared address has bit 0x80000 set.

The further Boolean expression represented at this decision imposes no additional restriction. This adjustment affects only the argument to F-0003; it does not change the already-completed selector-relative halfword write.

Software interrupt 0xDF follows F-0003. If execution continues through the represented post-interrupt path, F-0004 precedes F-0005. A halfword supplied by F-0005 is written at retained-interrupt-destination offset 0x2C. Invalid-instruction processing follows.

Neither the software interrupt nor invalid-instruction processing establishes a recoverable error or a resumable boundary.

### Negative discriminator: selected-record processing

A negative discriminator selects record processing. The selected address’s low halfword is first written at selected-address offset 0x2E.

F-0006 supplies an input to F-0007. Between those interactions, the mixed address is written at prepared-address offset 0x38. F-0007 also receives the stack-associated word at displacement 0x264; the capture time of that word is unresolved.

After F-0007, a word is sampled at selected address plus the shared displacement. A nonzero displacement chooses direct completion; zero chooses replacement. It is unresolved whether the sampling address and branch decision use one coherent observation of the displacement.

Direct completion does not publish. Its completion identity is:

| Completion role | Address or value |
|---|---|
| Primary address | Selected address plus four |
| Secondary address | Selected address |
| Auxiliary address | Selected address |
| Completion input | Sampled word |
| Interaction | F-0020 |

### Replacement identity

Replacement first passes the sampled word to F-0008. After F-0008 returns, F-0009 receives the words at offsets 0x00 and 0x04 of the embedded replacement-input object. The relative order of these two reads is unresolved.

Replacement identity is obtained only after F-0009, from the selected record:

| Read order | Offset | Interpretation |
|---|---|---|
| First | 0x04 | Replacement primary address |
| Second | 0x0C | Replacement state |
| Third | 0x10 | Value from which 0x5A is subtracted to obtain the replacement secondary address |

Thus, effects of F-0008 or F-0009 can change the identity used by the remaining route. No earlier snapshot of these fields is established.

The replacement secondary address’s low halfword is written at replacement-primary offset 0x3E. F-b872f3fbd55f then receives replacement state plus 0x24 and supplies a pair. The upper component’s low byte is written at twice the replacement primary address.

Replacement publishes using these replacement addresses and state, with the embedded replacement-descriptor address as descriptor. Its publication pair is the returned pair with 0x54 added only to the lower component.

### Construction and overflow decision

Construction obtains its record address by reading a word at the F-0001 lookup base plus 0x48. Its descriptor is the inherited construction state doubled and truncated to 32 bits.

Before initialization, the initial secondary address is written as a word at construction-record address plus the preliminary-write displacement. The overflow decision is captured after this write and before initialization. It tests signed 32-bit overflow from adding 0x32 to the truncated double of the initial secondary address. Overflow during doubling is not the tested condition.

Both overflow outcomes initialize the same construction-record layout, in ascending offset order:

| Offset | Width | Installed value |
|---|---|---|
| 0x00 | Word | Embedded construction-tag address |
| 0x04 | Word | Construction-record address |
| 0x08 | Word | Construction descriptor |

These writes may overwrite the preliminary write. F-0010 follows initialization, and F-0011 then supplies a pair. Their effects do not cause the captured overflow decision to be recomputed.

Overflow selects publication with the initial primary and secondary addresses, the inherited construction state, the construction descriptor, and the unchanged F-0011 pair. The construction-record address does not replace either publication address.

Without overflow, F-0012 is followed by F-0013, which receives the address of stack-associated storage at displacement 0x29C. F-0014 is then reached through the unresolved continuation destination. If it returns, an ordinary return is represented. No explicit publication or completion occurs on this route within F-afd60aafb68b.

### Publication effects

Replacement and overflow construction share a publication layout. Their primary address, secondary address, state, descriptor, and pair come from the route-specific rules above.

| Storage | Width | Published value |
|---|---|---|
| Numeric address 0x55 | Word | Value associated with 0x69, converted to unsigned 32-bit form |
| Primary-address offset 0x2C | Halfword | Descriptor’s low sixteen bits |
| Numeric address 0x4D | Word | Primary address |
| Numeric address 0x51 | Word | Secondary address |
| Numeric address 0x59 | Word | State |
| Numeric address 0x5D | Word | Descriptor |

The writes occur in table order. The value associated with 0x69 is obtained for the first write; its source width and signedness are unresolved. The primary-relative halfword can overlap the fixed-address fields.

After publication, the route-specific interaction receives two transformed pair components: the signed lower component arithmetically shifted right five bits, and the upper component plus 0x55 with wrapping. Replacement uses F-0015; construction uses F-0019. Their equivalence is not established.

F-0016 subsequently supplies a completion input. Only after it returns is the value associated with numeric location 0x76 obtained and converted to an unsigned 32-bit auxiliary address. Its source width and signedness are unresolved. Effects of F-0016 can therefore affect this later observation.

Publication-derived completion retains the published primary and secondary addresses and uses F-0017.

### Completion effects and terminal boundary

Both completion routes retain the word at primary-address offset 0x37 before invoking their completion interaction. F-0017 or F-0020 receives the completion input and the embedded completion-argument address.

After that interaction, the secondary record receives four word writes in ascending offset order:

| Offset | Installed value |
|---|---|
| 0x00 | Additional machine-state output available after the interaction |
| 0x04 | Auxiliary address |
| 0x08 | Previously retained word |
| 0x0C | Auxiliary address |

The first field is not established as an ordinary return value. The retained word is not reread after the interaction.

Undefined-instruction processing follows with represented operands 0xD3 and 0x085710D6. Its represented result supplies the destination for F-0018. Whether this exceptional operation permits continuation or provides a usable destination is **documented_only**. No later behavior is established.

## Side effects

The function’s established effects are its specified memory accesses, dependency interactions, conditional coprocessor activity, and exceptional control transfers.

Aliasing is not excluded among records, embedded objects, fixed-address storage, and sampled fields. Ordering therefore matters even without concurrent execution. Earlier writes may affect later reads, while retained values remain the values already observed.

No explicit allocation or release is established. Passing an address does not establish ownership transfer, permission to retain it, accessible extent, or lifetime extension.

Opaque dependencies may modify memory or machine state, allocate or release storage, fault, or fail to return. Their behavior is not constrained by an assumed preservation rule. In particular, later observations of externally stored values must not be replaced by assumed pre-interaction snapshots.

There is no established synchronization, memory barrier, atomic publication, rollback, or concurrency protocol. Fault handling and externally imposed memory behavior remain unknown.

## Dependencies

Opaque IDs identify interaction sites; distinct IDs do not necessarily imply distinct implementations.

| ID | Established contribution |
|---|---|
| F-0001 | Supplies the construction lookup base |
| F-0002 | Consumes a halfword payload and supplies the preparation pair |
| F-0003 | Consumes the interrupt-route selector argument |
| F-0004, F-0005 | Ordered post-interrupt interactions; F-0005 supplies a halfword |
| F-0006, F-0007 | Produce and consume the selected-record processing input |
| F-0008, F-0009 | Process replacement inputs before replacement identity is read |
| F-b872f3fbd55f | Consumes replacement state plus 0x24 and supplies a pair |
| F-0010, F-0011 | Follow construction initialization; F-0011 supplies a pair |
| F-0012, F-0013, F-0014 | Form the non-overflow construction continuation |
| F-0015, F-0019 | Replacement and construction publication interactions |
| F-0016 | Supplies publication-derived completion input |
| F-0017, F-0020 | Publication-derived and direct completion interactions |
| F-0018 | Result-directed interaction beyond the terminal exceptional boundary |

Unspecified arguments, hidden inputs, memory effects, preservation, failures, ownership, retention, and returnability are **documented_only**.

## Acceptance examples

These are conditional static consequences, not executed tests. Each assumes accessible reached storage and continuation through preceding interactions.

1. **Payload consumption.** An initial primary address of 0x00020000 gives prepared address 0x00040000 and mixed address 0x00060000. A halfword of 0x1234 at 0x000C0026 is the payload supplied to F-0002 with 0x20. Argument-width extension remains unresolved.

2. **Truncated classification.** Initial primary address 0x00020000 produces discriminator 0x80000000 and selects selected-record processing. Address 0x00040000 produces zero and selects interrupt processing without the conditional coprocessor interaction.

3. **Selector argument differs from write destination.** Initial primary address 0x00040001 produces a positive discriminator and prepared address 0x00080002. For selector 0x20, the selector-relative halfword write begins at 0x5E, while F-0003 receives 0xFFFFFEB0.

4. **Overflow boundary.** Initial secondary address 0x3FFFFFE6 doubles to 0x7FFFFFCC; adding 0x32 does not overflow. Address 0x3FFFFFE7 doubles to 0x7FFFFFCE; adding 0x32 does overflow. Both initialize construction storage and reach F-0010 and F-0011; only the latter selects publication. Address 0x80000000 doubles to zero and selects the non-overflow continuation.

5. **Independent pair components.** A replacement lower component of 0xFFFFFFF0 becomes 0x00000044 after its 0x54 adjustment, without incrementing the upper component. A publication lower component of 0xFFFFFFE0 yields first-argument bit pattern 0xFFFFFFFF after arithmetic right shift.

6. **Late replacement identity.** If F-0009 changes selected-record offsets 0x04, 0x0C, or 0x10, replacement uses the values subsequently read there, not earlier values.

7. **Unresolved displacement coherence.** If sampling observes displacement twelve but the branch observes zero, replacement receives the word sampled at selected-address offset twelve. This is conditional behavior, not evidence that two physical observations occur.

8. **Construction storage is not publication identity.** A preliminary-write displacement of zero allows tag initialization to overwrite the preliminary write. Overflow still publishes the initial primary and secondary addresses, not the construction-record address.

9. **Completion retains its observation.** With direct-completion selected address 0x2000, the retained word is read at 0x203B. F-0020 changing that storage does not change the word later written at secondary offset 0x08. Secondary offsets 0x04 and 0x0C receive 0x2000.

10. **Publication overlap.** Primary address 0x29 places the descriptor halfword at 0x55, overlapping the preceding word write. The later halfword replaces the overlapping bytes. Their numeric interpretation depends on byte order.

## Uncertainties

The following remain **documented_only**:

- The full calling convention and initialization of inherited inputs.
- Capture timing for the entry-selection byte and stack-associated word.
- Dependency implementations, preservation, failures, retention, ownership, and returnability.
- Extension of the preparation halfword argument.
- Source widths and signedness associated with 0x69 and 0x76.
- Observation coherence and physical-read count for the shared displacement.
- Relative read order of the two embedded inputs supplied to F-0009.
- The calling-convention meaning of completion’s first installed word.
- Storage validity, alignment behavior, byte order, external interference, and device-like memory behavior.
- Embedded-object contents, extent, mutability, lifetime, and application meaning.
- Coprocessor behavior and exceptional-operation resumability.
- F-0014’s continuation semantics and the reliability of its represented return boundary.
- F-0018’s reachability and destination validity.

These unknowns do not establish recovery, preserved globals, successful completion, or ownership transfer.

## Evidence

**Established static evidence:** The supplied function representation supports the specified access widths, layouts, relative memory-effect order, branch conditions, arithmetic transformations, interaction-site distinctions, and exceptional boundaries.

**Derived consequences:** The acceptance examples follow from 32-bit truncation and wrapping, signed classification, signed-addition overflow, independent pair components, arithmetic right shift, and ordered overlapping accesses.

**Inferred intent:** “Preparation,” “construction,” “publication,” and “completion” are descriptive groupings. They do not establish an application object model, lifecycle, allocation policy, or synchronization protocol.

**Evidence limits:** Dependency implementations and environmental definitions are unavailable. Prior prose, comments, and reviewer feedback are not independent behavioral evidence. Represented exceptional continuations do not prove executable resumability. No runtime tests have been performed.
