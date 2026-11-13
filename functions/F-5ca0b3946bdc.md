<!-- Trusted provenance: F-5ca0b3946bdc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ca0b3946bdc

## Purpose

F-5ca0b3946bdc performs an ordered sequence of mixed-width memory reads and writes, prepares local storage, and invokes an indirect dependency. If that dependency returns, it changes additional represented storage and calls F-cdec09f5d432. If that call returns, execution reaches a represented nonreturning terminal operation.

No application-level purpose, object model, allocation protocol, or ownership transfer is established. Address-like values are copied, narrowed, added, multiplied, and followed through memory; those operations do not establish that the affected storage contains valid objects.

This contract distinguishes:

- **Established:** directly supported by the supplied static body.
- **Inferred intent:** interpretation without contractual force.
- **documented_only:** unresolved behavior that must not be replaced by an assumption.

Every operation below is conditional on execution reaching it. Invalid accesses, unresolved arithmetic semantics, the unresolved byte, and dependency behavior can prevent completion.

## Interface

### Explicit inputs and output

The represented interface has seven positional inputs and no ordinary result.

| Input | Width | Established use |
|---|---:|---|
| First | Signed 32 bits | Address base, stored value, and shift operand. |
| Second | 32 bits | No explicit body use. |
| Third | 32 bits | No explicit body use. |
| Fourth | 8 bits | Early byte-store source. |
| Fifth | 32 bits | Its argument-storage address is passed to the indirect dependency. |
| Sixth | 32 bits | No explicit body use. |
| Seventh | 32 bits | Its represented argument storage is cleared after the indirect dependency returns. |

Lack of explicit use does not establish irrelevance to an incompletely represented calling convention. Writing argument storage does not establish modification of caller-owned storage.

### Implicit inputs

Four entry-state addresses are designated here as:

- **Anchor:** initial offset-68 replacement destination.
- **Feeder:** initial offset-68 read source.
- **Workspace:** initial self-address and byte-store destination.
- **Companion:** address written through the first input and later used as a destination.

Another entry-state value supplies the indirect call target.

Ten global storage sources supply successive address selections. **Each selection involves two separate global evaluations:** the first supplies a retained selected address; the second supplies the value written at an offset-68 destination. Equality between those evaluations is not established.

A separate global evaluation supplies the first argument to F-cdec09f5d432.

One represented byte is consumed before any assignment to its storage. Its value and execution consequences are **documented_only**.

### Widths and memory requirements

Offsets are byte offsets. A **word** is 32 bits, a **halfword** is 16 bits, and a **byte** is 8 bits. Halfword reads described below are signed and contribute sign-extended values. Loaded plain-character bytes have unresolved signedness when promoted.

Fixed offsets include 1, 9, 13, 17, 20, 21, 29, 68, 84, 100, and 116. Other accesses use sums or products directly as addresses. These accesses do not establish a common structure layout.

For a common base, offset 21 lies inside the word starting at offset 20. Endianness determines its numeric significance. All other overlap depends on actual addresses.

There are no represented null, bounds, alignment, permission, or arithmetic checks. Each reached access requires storage valid for its exact width and direction. Valid storage at the first input alone is insufficient.

## Behavior

### Ordering and terminology

The contract preserves access order, not merely final memory contents. Earlier writes can affect later reads through aliasing, including reads of global storage. Retained values are distinct from subsequent rereads of their sources. Byte and halfword writes do not authorize wider writes.

The following descriptive roles identify retained values, not public fields or source identifiers:

- **Working address:** the address used for recurring offset-17 and self-address stores.
- **Carried word:** a word retained for installation at the next selected address’s offset 20.
- **Replacement destination:** the base whose offset 68 receives the next global source’s second evaluation.

Each numbered selection below starts with this established ordering:

1. Evaluate its global source and retain the selected address.
2. Evaluate that same global source again and write this second result at the specified replacement destination’s offset 68.
3. Write the incoming carried word at the retained selected address’s offset 20.

The two evaluations are separate observations. The second value is not substituted for the retained address in subsequent accesses. Across ten selections, this accounts for **twenty global evaluations**, apart from the later global argument evaluation for F-cdec09f5d432.

The selection descriptions specify the remaining accesses and how values pass between phases.

### Entry preparation and first selection

First, read the feeder’s offset-68 word. Then write the workspace address as a word at workspace offset 84.

The first selection uses the anchor as replacement destination and the feeder sample as incoming carried word. After its three selection-start accesses:

- Write the fourth input at workspace offset 17.
- Take the **first** word sample from first-input offset 100.
- Read the workspace’s offset-68 word, then a signed halfword at workspace plus the retained selected address.
- Write the companion address at first-input offset 84.
- Read the selected address’s offset-20 word; this becomes the working address.
- Write the selected address’s low byte at working-address offset 13.
- At the sum of the signed-halfword value and selected address, write the low 16 bits of the workspace offset-68 sample.
- Read the selected address’s offset-68 word as the next carried word.
- Write the working address to its own offset 84.

The workspace offset-68 sample becomes the second selection’s replacement destination.

### Second selection

After the selection-start accesses:

- Write the low byte of the first offset-100 sample at working-address offset 17.
- Read the working address’s offset-68 word, then a signed halfword at working address plus the selected address.
- Write the companion address at first-input offset 84 again.
- Read the selected address’s offset-20 word as the new working address and write the selected address’s low byte at its offset 13.
- Read a replacement address through the **incoming carried word’s offset 20**.
- Write the first input at companion offset 84.
- Write the low 16 bits of the earlier working-address offset-68 sample at signed-halfword value plus this replacement address.
- Read this replacement address’s offset-68 word as the next carried word.
- Write the new working address to its own offset 84.

The earlier working-address offset-68 sample becomes the third selection’s replacement destination.

### Third selection

After the selection-start accesses:

- Write the low byte of twice the working address at its offset 17.
- Read a signed halfword at working address plus selected address.
- Read a plain-character byte at signed-halfword value plus incoming carried word.
- Write the companion address’s low byte at selected-address offset 29.
- Write the promoted character value, narrowed to 16 bits, at signed-halfword value plus selected address.
- Read selected-address offset 68 as the next carried word.
- Write the working address to its own offset 84.

The promoted character value becomes the fourth selection’s replacement destination. Its signedness is unresolved.

### Fourth selection

After the selection-start accesses:

- Write the low byte of twice the working address at its offset 17.
- Read a signed halfword at working address plus selected address, then a plain-character byte at that halfword value plus incoming carried word.
- Take the **second** first-input offset-100 word sample and copy the full word to signed-halfword value plus offset 84.
- Write the incoming carried word’s low byte at promoted-character address plus offset 13.
- Read and retain selected-address offset 100. This separate word is called the **persistent sample** below.
- Write the promoted character, narrowed to 16 bits, at signed-halfword value plus selected address.
- Read selected-address offset 68 as the next carried word.
- Write the working address to its own offset 84.

The promoted character becomes the fifth selection’s replacement destination. The persistent sample is distinct from all six samples through first-input offset 100.

### Fifth selection

After the selection-start accesses:

- Write the low byte of twice the working address at its offset 17.
- Take the **third** first-input offset-100 sample.
- Read a signed halfword at working address plus selected address, then a plain-character byte at halfword value plus incoming carried word.
- Write the selected address’s low byte at first-input offset 1.
- Write the incoming carried word’s low byte at selected-address offset 13.
- Write the persistent sample as a word at selected-address offset 68.
- Write the selected address’s low byte at promoted-character address plus offset 13.
- Write the promoted character, narrowed to 16 bits, at halfword value plus selected address.
- Read selected-address offset 68 as the next carried word.
- Write the working address to its own offset 84.

The final offset-68 read occurs after intervening writes; it is not an unconditional reuse of the persistent sample. The promoted character becomes the sixth selection’s replacement destination.

### Sixth selection

After the selection-start accesses:

- Write the low byte of the third first-input offset-100 sample at working-address offset 17.
- Take the **fourth** first-input offset-100 sample.
- Read working-address offset 68.
- Read a signed halfword at working address plus the **retained selected address**.
- Read a replacement address through incoming carried word plus offset 116.
- Read the old working address’s offset-20 word as the new working address.
- Write the replacement address’s low byte at the earlier offset-68 sample’s address plus offset 13.
- Write the low 16 bits of that offset-68 sample at halfword value plus the **replacement address**, not plus the selected address.
- Read replacement-address offset 68 as the next carried word.
- Write the new working address to its own offset 84.

The earlier working-address offset-68 sample becomes the seventh selection’s replacement destination.

### Seventh selection

After the selection-start accesses:

- Write the fourth first-input offset-100 sample’s low byte at working-address offset 17.
- Read working-address offset 68, then a signed halfword at working address plus selected address.
- Write the selected address’s low byte at incoming carried word plus offset 21.
- Write the incoming carried word as a word at **selected-address offset 68**.
- Only then read a replacement address through the earlier working-address offset-68 sample plus offset 116.
- Write that sample’s low 16 bits at halfword value plus replacement address.
- Read replacement-address offset 68 as the next carried word.
- Write the working address to its own offset 84.

The working-address offset-68 sample becomes the eighth selection’s replacement destination. The offset-21 and selected-address offset-68 stores precede the replacement-address read and may affect it through aliasing.

### Eighth selection

After the selection-start accesses:

- Write the low byte of twice the working address at its offset 17.
- Take the **fifth** first-input offset-100 sample.
- Read working-address offset 68, then a signed halfword at working address plus selected address.
- Read a new working address through signed-halfword value plus offset 20.
- Write the offset-68 sample to its own address plus offset 84.
- Write the persistent sample as a word at selected address plus incoming carried word.
- Replace selected-address offset 20 with the offset-68 sample.
- Read the word at persistent-sample address plus offset 68 and write its low byte at selected-address offset 13.
- Write the offset-68 sample’s low byte at new-working-address offset 13.
- Write its low 16 bits at signed-halfword value plus selected address.
- Read selected-address offset 68 as the next carried word.
- Write the new working address to its own offset 84.

The offset-68 sample becomes the ninth selection’s replacement destination. Both full-word stores preceding the persistent-sample-address read can affect that read through aliasing.

### Ninth selection

After the selection-start accesses:

- Write the fifth first-input offset-100 sample’s low byte at working-address offset 17.
- Take the **sixth** first-input offset-100 sample.
- Read working-address offset 68, then a signed halfword at working address plus selected address.
- Write the working address at incoming carried word plus offset 84.
- Write the sixth sample’s low byte at working-address offset 17.
- Write the **retained selected address** at the earlier offset-68 sample’s address plus offset 68. This is not another global evaluation.
- Read a new working address through signed-halfword value plus offset 20.
- Write the offset-68 sample to its own address plus offset 84.
- Write the persistent sample at selected address plus incoming carried word.
- Replace selected-address offset 20 with the offset-68 sample.
- Read persistent-sample address plus offset 68 and write that word’s low byte at selected-address offset 13.
- Write the offset-68 sample’s low byte at new-working-address offset 13, then its low 16 bits at halfword value plus selected address.
- Read selected-address offset 68 as the next carried word.
- Write the new working address to its own offset 84.

The offset-68 sample becomes the tenth selection’s replacement destination.

### Tenth selection and arithmetic-dependent accesses

After the selection-start accesses, write the sixth first-input offset-100 sample’s low byte at working-address offset 17. Read working-address offset 68, then a signed halfword at working address plus selected address.

Perform two consecutive identical pairs: write the signed-halfword value as a word at its own address plus offset 84, then write the working address’s low byte at selected-address offset 17. Both pairs are represented accesses.

The deeper chain starts at the product of the signed-halfword value and the working-address offset-68 sample. Read that product address plus offset 68; follow the resulting word through offset 116, then follow that result through offset 20. Read this final address’s offset-20 word and retain it before replacing the same word with twice the working address.

Read the retained word’s offset-20 word and multiply that result by the signed-halfword value. This is the **first later product**. Write the halfword value to its own address plus offset 84, then write the retained word’s low byte at selected-address offset 17.

Read and retain the first later product’s offset-116 word as the **late retained result**. Multiply the first later product by the signed-halfword value to obtain the **second later product**. Then, in order:

- Repeat the halfword-derived self-address store.
- Write the unresolved byte at selected-address offset 17.
- Read second-later-product offset 68.
- Read first-input offset 116.
- Write the late retained result at selected-address offset 100.
- Write the sampled product word’s low byte at selected-address offset 9.
- Reread second-later-product offset 68 and write its low byte at second-later-product offset 17.
- Repeat the halfword-derived self-address store.
- Define the previously unresolved byte’s represented storage from the late retained result’s low byte, and write that byte at twice the second later product plus offset 17.
- Read offset 68 at the product of the signed-halfword value and second later product.
- Read first-input offset 20.
- Write the earlier first-input offset-116 sample’s low byte at late-retained-result address plus offset 1.
- At the address obtained from first-input offset 20, write the latest product-derived sample’s low byte at offset 17, then the low byte of twice the second later product at offset 9.

The later assignment does not determine the earlier unresolved-byte store.

### Integer semantics

Byte stores retain eight bits; halfword stores retain sixteen bits. Signed halfword −1 contributes −1, not 65,535. A plain-character byte of 0xFF may promote differently depending on unresolved character signedness.

The represented arithmetic uses 32-bit integer operands. Defined modulo-\(2^{32}\) behavior is not established for signed multiplication, addition, or left shift. Negative shift operands, overflow, and address wrapping remain **documented_only**.

### Local preparation and continuation

Before the indirect call, represented local storage receives, in order: one first-input value shifted left four bits; one shifted left eight bits; four zero words; another occurrence of the four-bit shift; another occurrence of the eight-bit shift; a copy of the late retained result; another eight-bit-shift occurrence; another late-result copy; and another eight-bit-shift occurrence.

Thus there are two four-bit-shift occurrences, four eight-bit-shift occurrences, three late-result occurrences including its retained position, and four zeros. This does not establish a contiguous public argument block.

The indirect dependency receives the fifth input’s storage address and two zeros.

If it returns, the represented writes occur in this order: clear seventh-input storage; write marker 0x08400E1C to another 32-bit position; clear three additional 32-bit positions.

F-cdec09f5d432 then receives a newly evaluated global value and three zeros. If it returns, a terminal facility is invoked with represented operands zero and 0x08401154, followed by a represented indirect transfer through its result. This path is marked nonreturning.

## Side effects

Writes reach input-derived, entry-state-derived, global-derived, narrow-derived, and arithmetic-derived addresses, plus represented local and argument storage.

No allocation, release, reference-count change, transaction, rollback, cleanup, or synchronization protocol is established. Earlier successful writes have no represented rollback if a later access fails.

Exposing the fifth input’s storage address does not establish ownership transfer, adjacent readable extent, or permission to retain the address beyond its lifetime.

No dependency is known to preserve globals, reachable memory, or resources. No dependency is established to own or release an allocation.

## Dependencies

| Dependency | Established call-site behavior | Unresolved contract |
|---|---|---|
| Unidentified indirect dependency; no supplied opaque ID | Called after mutations and local preparation with the fifth input’s storage address and two zeros. | Calling convention completeness, effects, retention, ownership, and return behavior. |
| F-cdec09f5d432 | Called after the first dependency returns and post-call writes complete; receives a freshly evaluated global and three zeros. | Argument meaning, effects, ownership, exceptional exits, and return behavior. |
| Unidentified terminal facility; no supplied opaque ID | Reached after F-cdec09f5d432 returns; participates in the marked nonreturning path. | Exact termination, trap, transfer, and result semantics. |

No opaque identity is invented for an unidentified dependency.

## Acceptance examples

These are conditional criteria, not executed tests.

- **Distinct global evaluations:** If a selection’s first evaluation yields one address and its second yields another word, addressing uses the first value while the offset-68 replacement stores the second. Reusing either observation for both purposes is not established as equivalent.
- **Early overlap:** If workspace offset 17 overlaps first-input offset 100, the first sample occurs after the fourth-input byte store.
- **Sixth-selection replacement:** If the offset-116 load yields an address different from the selected address, the halfword destination uses the loaded replacement. The earlier halfword read still uses the selected address.
- **Seventh-selection ordering:** If the selected-address offset-68 store aliases the later offset-116 source, that later read observes memory after the store.
- **Persistent-sample aliasing:** In selections eight and nine, a summed-destination store or offset-20 replacement can change the subsequent read through the persistent sample. An earlier snapshot is insufficient.
- **Repeated observations:** The six first-input offset-100 samples need not agree. Likewise, the two second-later-product offset-68 reads remain distinct across the intervening byte store.
- **Width preservation:** An 0xAB byte store changes only its destination byte. The fourth-selection offset-100 copy transfers all 32 bits.
- **Signed narrow value:** A signed halfword representing −1 contributes −1 to addressing and multiplication. Access validity remains conditional.
- **Duplicate writes:** Both identical self-address/byte-store pairs in the tenth selection remain in the represented sequence.
- **Dependency continuation:** If the first dependency does not return, no post-call clearing, marker write, later call, or terminal operation is reached. If it changes the later call’s global source and returns, the later call uses its subsequently evaluated value.

## Uncertainties

The following remain **documented_only**:

- Completeness of the interface and implicit entry state.
- Address provenance, validity, aliasing, and access properties.
- Global stability across evaluations.
- Unresolved-byte value and consequences.
- Character signedness, endianness, alignment, and unaligned-access behavior.
- Signed arithmetic overflow, shift validity, and wrapping.
- Atomicity, concurrency visibility, volatility, and device-memory behavior.
- Physical local/argument layout and exposed extent.
- Dependency effects, lifetimes, ownership, and exceptional exits.
- Exact terminal behavior.

**Inferred intent:** repeated address-field mutations might manipulate links or execution state. Neither interpretation establishes a semantic data model.

Notices concerning omitted unreachable regions and overlapping global symbols limit completeness and typing confidence; they establish no additional reachable effects.

## Evidence

This contract uses only the supplied static body. The prior draft and reviewer feedback are not independent behavioral evidence.

Established evidence includes the mixed-width accesses; twenty global evaluations across ten selections; retained-versus-reread distinctions; six first-input offset-100 samples; selection-specific replacement-address chains; repeated writes; arithmetic-derived accesses; local value multiplicities; ordered post-call writes; dependency order; and marked nonreturning terminal path.

No dependency bodies or external implementation were accessed. No runtime tests have been performed.
