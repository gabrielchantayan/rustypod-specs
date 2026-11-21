<!-- Trusted provenance: F-e54b6eefded4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e54b6eefded4

## Purpose

F-e54b6eefded4 accepts three explicit 32-bit arguments, invokes an ordered sequence of indirect dependencies, performs mixed-width memory mutations, and ends with an unresolved indirect invocation.

**Established static evidence:** the supplied representation contains dependency-result forwarding, packed-field extraction, address arithmetic, repeated stores, fixed-storage reads and writes, and a final invocation with four represented argument words. No explicit validation, rollback, or error-status return is shown.

**Inferred intent:** some accesses resemble updates to interconnected records. Literal values include values that can represent characters. Neither observation establishes a record-management, formatting, rendering, or serialization purpose.

**Contract boundary:** this document specifies the supported static observations and conditional consequences. It does not establish that the represented control flow is executable on a particular target. Dependency implementations, memory mappings, and the actual calling convention remain **documented_only**.

## Interface

### Explicit arguments and result

| Item | Width and representation | Established use |
|---|---|---|
| First argument | Signed 32-bit integer | Forwarded unchanged to the first invocation and also supplied after a left shift of 14 positions. |
| Second argument | Opaque 32-bit word | Forwarded unchanged to the first invocation. |
| Third argument | Opaque 32-bit word | Forwarded unchanged to the first invocation. |
| Return | No explicit result value | No status, count, or result address is established. |

The first invocation has four explicitly represented argument words. This does not make the entry interface a four-argument interface.

### Environmental inputs

Behavior additionally depends on:

- Incoming execution state that supplies the first indirect destination.
- Two incoming unsigned 32-bit quantities subjected to logical right shifts by 15.
- Secondary result-channel values associated with earlier invocations.
- Dependency results represented as 32-bit words or paired 64-bit quantities.
- Contents of computed and fixed-address storage.
- A fixed-storage word that supplies a later address base.
- The unresolved final invocation destination.

These inputs are not established portable parameters. Identical explicit arguments need not produce identical effects.

### Preconditions for successful traversal

Every reached invocation destination must be executable under the actual calling convention. Every reached memory location must support the required width, alignment, and access permissions.

No explicit null, bounds, alignment, permission, or ownership checks are represented. Zero arguments do not select a demonstrated no-op path. Address-like values obtained from bytes or words are used without demonstrated validation.

## Behavior

### Dependency processing and ordering

A long sequence of indirect invocations precedes the first explicit data-memory store in this body. Dependencies may themselves access or mutate memory during that sequence.

The first invocation receives the three entry arguments and the first argument shifted left by 14. Later invocations receive combinations of zero, earlier results, extracted fields, shifted values, incoming execution state, and secondary result-channel values.

Invocations without explicit argument expressions do not establish cleared argument registers or an empty machine-state interface. Likewise, an unused ordinary return value does not make an invocation dispensable: its other effects are unknown.

Invocation order is part of the represented behavior. Similar argument patterns do not establish identical destinations or interchangeable calls. The exact executable identities and complete per-invocation contracts remain **documented_only**.

### Widths and arithmetic

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit paired results. Lower and upper 32-bit portions of paired results are consumed independently. This establishes a logical split, not target byte order or a confirmed scalar-return convention.

Representative unsigned field operations are:

| Transformation | Selected information |
|---|---|
| Mask to 24 bits, then shift right by 8 | Original bits 8–23; range 0–65,535. |
| Mask with 0x0003FFFF, then shift right by 11 | Original bits 11–17; range 0–127. |
| Mask with 0x0001FFFF, then shift right by 4 | Original bits 4–16; range 0–8,191. |
| Shift left by 8, mask with 0x00007FFF, then shift right by 6 | Under 32-bit bit-vector semantics, four times the original low seven bits. |
| Mask with 0x0000FFFF, then shift right by 5 | Original bits 5–15, subject to any earlier transformations. |

Left shifts and power-of-two multiplication also produce addresses and invocation arguments.

Unsigned 32-bit transformations retain only the low 32 bits. Several transformations instead have signed operands. Portable behavior is not established for overflowing signed multiplication, negative signed left shifts, or signed address arithmetic. Any wrapped interpretation below is conditional on 32-bit bit-vector behavior.

Width must be preserved across successive transformations. Two successive left shifts by 14 retain at most the original low four bits under that interpretation. Successive shifts by 14 and 23 produce zero. Explicitly cleared working values also produce zero-valued derivatives; this says nothing about dependency preservation of globals.

### Initial direct-memory phase

The following descriptive bases distinguish address provenance without assigning object semantics:

- **Packed base:** an earlier result shifted left by 25.
- **Large scaled base:** another result multiplied by 0x02000000.
- **Small base:** twice the field produced by the left-eight, mask-0x7FFF, right-six transformation.
- **Final paired result:** the last paired result preceding direct writes.

A word store at the upper portion of the final paired result plus 0x14 precedes the first byte store at packed base + 9. A word containing the large scaled base is then stored at 256 times the lower result portion plus 0x54, before a second byte store at packed base + 9.

The large scaled base’s offset-0x14 word is cleared before its offset-5 byte is captured. The small base’s offset-0x14 word is cleared before that byte is zero-extended into a word at large scaled base + 0x34. A subsequent read of large scaled base + 0x14 precedes another clearing of small base + 0x14 and two successive byte stores at small base + 0x15.

These accesses cannot be reduced to final assignments. Computed bases may overlap, including the word at offset 0x14 and byte at offset 0x15.

Under 32-bit bit-vector semantics, the small base ranges from zero through 1,016 in multiples of eight. Such low addresses are not evidence of allocated objects.

### Fixed-byte capture and linked-state updates

After the two byte stores at small base + 0x15, the body captures the byte at fixed address **0x05**, zero-extending it to a word. Only afterward does it write the low byte of the small base to fixed address **0x01**.

That captured byte supplies subsequent addresses:

- Its value plus 0x15 receives a byte.
- Its value plus 0x54 supplies a word that is captured before replacement.
- That same offset-0x54 word is then overwritten.

The captured byte is not refreshed after the write to 0x01 or intervening computed writes. If an intervening destination aliases 0x05, later address calculations in this chain still use the earlier capture.

The word captured from the byte-derived offset-0x54 location becomes a significant address-arithmetic input, called the **link root** below. Subsequent activity includes self-valued word stores, repeated byte stores, word values reused as addresses, and byte-derived accesses to offset-0x74 words.

Observed computed offsets include:

| Access width | Offsets represented |
|---|---|
| Byte | 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| Word | 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |
| Halfword | Separately scaled addresses |

These accesses do not establish one structure layout, allocation extent, or field meaning. Repeated reads separated by writes cannot generally be merged: the intervening destinations may alias the source.

### Literal overwrites and later address capture

A later byte-derived lookup supplies a word called the **late lookup value**. The body writes 0x3A as:

- A byte at four times that value.
- A halfword at eight times that value.
- A word at four times that value plus 0x20.

For the following phase, the **adjusted displacement** is four times the link root minus 0x96. The **adjusted base** is four times that displacement. Arithmetic qualifications above apply.

The word at adjusted base + 0x20 receives, in order:

1. 0x20.
2. 0x2F.
3. A zero-extended byte captured from adjusted base + 4.
4. 0x20 again.

The byte capture occurs after the first two stores and before the third. Matching only the final word does not preserve this access sequence.

After the final 0x20 store, the body reads the fixed address-supplying word represented at **0x087FBBBC**. Its captured value is the **publication anchor**. This read occurs before the word store of 0x20 at fixed address **0x60**.

Consequently, earlier literal writes can affect the anchor capture if storage aliases. Conversely, the subsequent write at 0x60 and later computed writes do not cause the anchor to be recaptured.

### Repeated publication groups

The publication suffix uses distinct values that must not be conflated:

| Label | Derivation | Role |
|---|---|---|
| Publication payload | Four times the late lookup value | Unchanged word payload throughout the groups. |
| Initial destination base | Eight times the link root | Destination base for the first group. |
| Adjusted destination base | Eight times the adjusted displacement | Destination base for following groups. |
| Adjusted base | Four times the adjusted displacement | Supplies one captured byte payload and a byte destination. |
| Publication anchor | Captured fixed-storage word | Supplies the other captured byte payload and two destinations. |

Each group has this observable order:

1. The publication payload is stored as a word at the group’s destination base + 0x54.
2. The captured low byte of the adjusted base is stored at publication anchor + 0x19.
3. The captured low byte of the publication anchor is stored at adjusted base + 0x11.
4. The group’s destination base is stored as a word at publication anchor + 0x34.

Thus, the two word payloads differ. The offset-0x54 payload remains unchanged. The offset-0x34 payload is the destination base: it changes from the initial destination base to the adjusted destination base after the first group.

The two byte payloads are captured values, not fresh memory reads. Aliasing can affect memory and later reads, but does not itself change those captures.

The suffix is finite and repetitive in the supplied representation. Its exact repetition count is retained as **documented_only**. Repetition must not be collapsed when access traces or intermediate observations matter.

### Final-phase captures and fixed-byte update

After the publication suffix, the first relevant captures and update have a specific order:

- The word at fixed address **0xA9** is captured.
- The word at publication payload + 0x74 is captured.
- A byte is written at fixed address **0x3A**.
- The word at twice the late lookup value + 0x14 is captured.
- The word at adjusted base + 4 is captured.

The byte written at 0x3A is derived from eight times the adjusted displacement plus 0x30, narrowed to eight bits. This is not an established character-formatting operation.

Not all these words are read before the 0x3A update. If that byte destination overlaps either later word source, those later captures must reflect the intervening write according to the target memory model and byte order.

### Captured 0xA9 word versus live 0x67 reads

Further computed writes include literal byte 0x6C at adjusted base + 8, byte updates at offsets 9 and 0x0D, and a self-valued word at an offset-0x54 destination. The low byte of the earlier 0xA9 capture is then written to fixed address **0x67**.

The body subsequently reads a computed offset-0x44 word before reading byte storage at 0x67 for a write to fixed address **0x21**. Later, it reads 0x67 twice more for separate byte stores at adjusted base + 9, with computed writes and reads between these uses.

These are distinct observations of byte storage at 0x67. They are not interchangeable with the earlier captured word from 0xA9, nor necessarily with each other. Intervening computed writes may alias 0x67. Separately, the full captured 0xA9 word is written to adjusted base + 0x44 before that destination is overwritten later.

### Variable shift and completion

A word loaded from the computed offset-0x44 source is explicitly represented as a **signed 32-bit integer**. It supplies a byte store at adjusted base + 0x10 and other word updates. It is also right-shifted by a count obtained from the low eight bits of the adjusted base; the low byte of the result is stored at twice the late lookup value + 1.

Operand signedness is established at the representation level. Behavior for negative operands, and for counts exceeding 31, remains unresolved. No portable result is promised for those cases.

The remaining phase performs interleaved captures and mixed-width mutations, including replacement of offset-0x10, offset-0x44, offset-0x54, and offset-0x74 words. Near completion:

- A word loaded through an offset-4 access supplies a halfword destination at twice its value; that destination receives 0x65.
- An offset-0x64 capture supplies a later word payload and a doubled final destination.
- A computed offset-0x54 word receives 100.
- A later offset-0x74 word is captured after that write.
- Another offset-0x64 capture supplies a base used for a word store at offset 0x54 and as the payload stored through the doubled final destination.

The final unresolved invocation receives four represented words: 0x72, the earlier offset-0x64 capture, the final-phase offset-0x14 capture described above, and the later offset-0x74 capture. A valueless return follows in the representation. Whether the invoked destination returns is unknown.

## Side effects

Direct accesses include one-, two-, and four-byte writes. Byte and halfword stores retain the low eight and sixteen bits respectively.

Meaningful fixed-storage effects include:

| Storage | Established direct activity |
|---|---|
| 0x01 | Byte write after the 0x05 capture. |
| 0x05 | Byte read supplying later addresses. |
| 0x15 | Byte write derived from the small base. |
| 0x21 | Byte write from a later read of 0x67. |
| 0x3A | Byte update between final-phase word captures. |
| 0x54 | Word write derived from the link root. |
| 0x60 | Word write of 0x20 after publication-anchor capture. |
| 0x67 | Byte write followed by multiple distinct byte reads. |
| 0xA9 | Word capture retained for later uses. |
| 0x087FBBBC | Word read supplying the publication anchor. |

No independence of storage regions is established. Computed accesses may overlap fixed storage or each other. The supplied overlap warning is not proof of any particular alias relationship.

No atomicity, locking, persistence, or transaction boundary is represented. Earlier effects may remain visible if a later invocation or access fails.

No allocation, release, reference-count operation, or ownership transfer is established for the body. Unknown dependencies may allocate, retain, or release memory; their behavior must not be assumed absent.

There is no explicit error-result or recovery contract. Invalid targets, inaccessible addresses, unsupported alignment, and arithmetic exceptions have no established handling path.

## Dependencies

F-e54b6eefded4 is the only supplied opaque function ID. No opaque IDs are available for the indirect dependencies; none are invented.

The dependency classes are an entry-state-derived initial destination, numerous subsequent indirect destinations, and the unresolved final destination. Their signatures, clobbers, memory effects, ownership effects, failure behavior, and return behavior remain **documented_only**.

Neither unused results nor calls without explicit arguments establish preservation of globals or execution state.

## Acceptance examples

These are conditional static checks, not executed tests.

1. **Initial forwarding.** With first argument 1 and supported 32-bit shift semantics, the first invocation receives 1, the unchanged second and third words, and 0x00004000.

2. **Field boundary.** Masking 0xAB123456 to 24 bits and shifting right by eight yields 0x1234. A contributing low-seven-bit value of 0x7F produces small base 1,016 under bit-vector semantics.

3. **Early fixed-byte capture.** If 0x05 contains 0x2A when read, the relevant later addresses include 0x3F and 0x7E. An intervening aliasing write that changes 0x05 must not redirect those uses of the captured value.

4. **Anchor timing.** If the address-supplying word contains an anchor immediately after the literal overwrite sequence, that is the value captured before the write at 0x60. A later overwrite of its backing storage does not refresh the anchor.

5. **Distinct publication payloads.** Suppose the link root is 100 and the late lookup value is 7. With safe arithmetic, the publication payload is 28, the initial destination base is 800, and the adjusted destination base is 2,000. The first group writes 28 at address 884 and 800 at anchor + 0x34. Following groups write 28 at address 2,084 and 2,000 at anchor + 0x34.

6. **Final capture ordering.** If the destination byte at 0x3A overlaps a later offset-4 word source, that capture occurs after the byte update. The earlier captures from 0xA9 and publication payload + 0x74 are not retroactively changed.

7. **Live fixed-byte reads.** If an intervening computed write changes 0x67 after its initialization from the 0xA9 capture, a subsequent read must use the changed byte. Substitution of the original captured low byte is invalid.

8. **Signed shift.** For a nonnegative signed operand 0x12345678 and count four, the shifted result is 0x01234567 and the stored byte is 0x67. Negative operands and excessive counts require an unresolved target contract.

9. **Failure and return.** An inaccessible derived destination establishes neither rollback nor successful return. If the final destination returns normally, the represented path returns without an explicit value.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and object semantics.
- Actual calling convention and implicit execution inputs.
- Dependency identities, contracts, and state preservation.
- Executability and completeness of the represented control flow.
- Memory mapping, byte order, alignment, and address wrapping.
- Signed overflow, negative signed-shift behavior, and excessive shift counts.
- Volatile, device-backed, shared, or concurrently observed storage.
- Allocation boundaries, lifetimes, ownership, and release obligations.
- Exact publication-suffix repetition count.
- Runtime reachability, failures, and final-destination return behavior.

These limitations prevent treating the three arguments as a safe, self-contained API.

## Evidence

This contract is based on the supplied static representation, not on the prior draft or reviewer assertions as independent evidence. Established observations include explicit input widths, result splitting, dependency ordering, mixed-width accesses, fixed-storage capture timing, distinct publication payloads, signedness of the variable-shift operand, and final invocation arguments.

Conditional arithmetic examples are deductions under their stated assumptions. No dependency bodies, target execution environment, runtime traces, or independent control-flow evidence were available. No runtime tests were performed.
