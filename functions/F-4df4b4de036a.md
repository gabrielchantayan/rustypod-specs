<!-- Trusted provenance: F-4df4b4de036a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4df4b4de036a

## Purpose

F-4df4b4de036a performs an ordered sequence of memory reads and mutations, derives further addresses from captured values, traverses and modifies linked storage, and finally invokes an unresolved indirect target with three arguments.

No application-level purpose is established. Character-sized constants, repeated offsets, and self-address stores do not establish text construction, object initialization, a container operation, or resource management.

This contract describes behavior supported by the supplied static representation. It does not establish a complete machine-level execution model. In particular, static-storage reads are interleaved with writes; they are not interchangeable with a snapshot taken at entry. Some detailed linked-access ordering remains documented_only below.

No runtime tests have been performed.

## Interface

### Explicit arguments

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the supplied body. |
| Second | 32 bits | Address arithmetic, doubling, and truncation into memory. |
| Third | 32 bits | Address arithmetic, including an early doubled value. |

The second and third arguments are represented as signed integers, but their use does not establish meaningful signed application types.

### Additional effective inputs

Execution also depends on:

- An incoming 32-bit register value.
- An incoming byte-sized register value.
- An unresolved indirect target, including its address bits.
- Static-storage values captured or consulted at specific points.
- A 32-bit read from absolute address `0xDB`.
- Memory reachable through computed addresses and 32-bit links.

The explicit arguments alone are insufficient to determine the effects.

### Output, completion, and preconditions

There is no represented return value. Outputs consist of memory changes and effects of the final indirect invocation. Normal return is represented after that invocation returns.

Successful completion requires every executed access to be valid under the execution environment and the final target to be callable. No validation, status result, error translation, rollback, allocation, or release is established.

### Widths and arithmetic

Explicit dynamic-memory accesses distinguish 8-, 16-, and 32-bit widths. Narrow stores retain the low 8 or 16 bits. A 32-bit store of `0x45`, for example, writes an entire 32-bit value.

Carry calculations explicitly use 32-bit unsigned arithmetic. Other expressions include signed addition, multiplication, shifts, and conversion between integers and addresses. Their machine-level overflow and address-wrap behavior is not fully established.

Plain-character signedness, byte order, alignment requirements, and physical load widths underlying narrowed static expressions remain unresolved. A static expression used through a byte or halfword conversion does not by itself prove a byte- or halfword-sized physical load.

## Behavior

### Entry and early effects

Before the first write, the routine captures a 32-bit static-storage value. That capture later supplies both a 16-bit store and a byte value. Moving this read after the initial write is not equivalent if the write aliases its storage.

The subsequent initial effects are ordered as follows:

- Write the low byte of the second argument at twice that argument.
- Read a 32-bit value at the second argument plus `0x52`.
- Write 16-bit `0x0052` at the sum of the second and third arguments.
- Write the low half of the captured dynamic word at that word’s value plus `0x52`.
- Write the low half of the unresolved target address at twice the second argument.

The first byte write therefore precedes the dynamic read, even though a later halfword write uses the same starting address.

The captured dynamic word is then stored through a doubled static-derived address. Further early writes combine that captured word with the incoming register values, overwrite a doubled address several times, and use a byte read from another doubled address to form a destination.

### Static-storage capture and reload boundaries

The labels below identify static storage by role and first appearance. They are not source identifiers, layout declarations, or claims that the locations are disjoint. Static symbols may overlap each other or computed destinations.

The table follows represented execution order. A “rereference” means a later static expression is present; it must not be replaced by an earlier capture without proving equivalence. Repeated references within a single expression do not establish a physical load count or evaluation order.

| Storage | Capture or rereference boundary | Subsequent significance |
|---|---|---|
| S01 | A 32-bit value is captured before the initial byte write. | Its low half is stored during the shift phase; its low byte is used in later writes. |
| S02 | Consulted after the initial target-derived halfword store. | Its doubled value addresses the following 32-bit store of the early dynamic word. |
| S03 | Captured after that 32-bit store; separately referenced in the destination of the next byte write. | The capture supplies later doubled destinations and address arithmetic. The separate reference supplies the destination added to the third argument. |
| S04 | Captured after that byte write and before the target-derived byte write at twice the captured S03 value. | Its low byte later replaces another byte at that doubled destination. |
| S05 | Captured after the target-derived byte write and before the next write there. | Supplies the `+0x2C` and doubled-byte-read addresses, and the unsigned comparison with `0x4E`. |
| S06 | Referenced after the S04-derived byte replacement: first for a full-value capture, then separately for a halfword capture. | The halfword capture is written at the captured S03 base plus `0x53` and later through S11. The full capture participates in the shift-address region. |
| S07 | Captured before the shift-source word is read. Referenced again after that read and the S08 capture. | The first capture supplies later doubled and summed destinations. Later references supply the doubled byte-store address and value. |
| S08 | Captured through a byte conversion after the shift-source word read and before the S07-referenced byte store. | Combined later with the target-address byte at twice the address of F-32c0bc2d3e1d. |
| S09 | Captured after the shifted result has been stored and a related halfword write completes. | Supplies a destination combined with the earlier address component and another destination combined with S06. |
| S10 | Consulted after the byte write at twice the address of F-32c0bc2d3e1d. | Supplies the destination for byte `0x59` at offset `+0x59`. |
| S11 | Captured after the subsequent function-address-associated write and halfword replacement at the shift-source address; referenced again for the next store destination. | The capture supplies the carry-sensitive doubling operand. The rereference supplies the base of the `+0x54` halfword store. |
| S12 | Captured after that halfword store. Referenced again for a comparison after the carry calculations, before their first write. | The capture supplies a later halfword value; the later reference supplies an unsigned-comparison result. |
| S13 | Captured immediately after the S12 capture. | Contributes to a later halfword sum. |
| S14 | Referenced during the carry calculations, after S12 and S13 are captured. | Supplies a carry-adjusted companion value and, through later references, the overflow test involving `0x59`. The latter test includes conditional evaluation. |
| S15 | Captured after the carry-region byte `0x45` store at a companion-derived address plus `0x9E`. | Supplies a `+0x45` destination and an ensuing `+0x74` link read. |
| S16 | Captured through a halfword conversion immediately after S15, before the next pointer-derived byte store. | Contributes to the halfword written at captured S15 plus `0x45`. |
| S17 | Captured near the terminal phase, after the linked-region constant writes and before reading the central `+0x74` link. | Supplies a doubled-address halfword value and a summed-address destination. |
| S18 | Captured after the terminal 32-bit `0x45` store and before the final byte `0x44` store. | Supplies the final doubled destination, the last halfword value, and the second indirect argument. |
| S19 | Captured immediately after S18 and before the final third-argument word is captured. | Supplies the 32-bit value written at twice captured S18. |

These boundaries matter under aliasing. For example, the store through S02 occurs before S03 is captured, and the store through the S11 rereference occurs before S12 and S13 are captured. Writes affecting those static locations can therefore change later inputs.

The 32-bit read at absolute address `0xDB` occurs in the linked region, not at entry. It follows two successive byte writes to a recovered base’s `+0x09` destination and precedes the next `+0x14` traversal from another recovered base. Its captured value is propagated later, including to `+0x54` destinations.

### Shift-derived addresses and overlapping updates

An early address component is twice the third argument plus the Boolean result of the unsigned comparison between captured S05 and `0x4E`.

A 32-bit word is read at that component plus captured S06. After the intervening S08 capture and S07-referenced byte store, the routine writes the low half of entry-captured S01 at the word’s starting address. It subsequently writes the captured target-address halfword at twice captured S07.

The original word, not a reload after those writes, supplies the signed 32-bit right-shift operand. Its low byte supplies the shift count. The result becomes both a stored 32-bit value and an address component.

Counts outside `0` through `31` have no established portable result. Negative signed operands also require execution-environment shift semantics. The contract does not substitute count masking, zero, or another guessed result.

The original word address is revisited by byte `0x52` and a halfword derived from the address component. These do not alter the already captured shift operand.

The address of F-32c0bc2d3e1d contributes to two byte-write destinations: twice that address, and that address plus the earlier component. No invocation of F-32c0bc2d3e1d is established.

### Carry-sensitive mutation region

Captured S11 is doubled using 32-bit unsigned carry semantics. A Boolean comparison between the incoming 32-bit value and the earlier address component adds either zero or one.

A carry indicator is the logical OR of carry from doubling and carry from adding that increment. It adjusts an S14-derived companion value.

Another address component combines captured S06, the doubled-and-incremented result, and an additional overflow indicator. That indicator concerns adding `0x59` and the preceding carry indicator to S14-derived values. The represented S14 references are not an entry snapshot.

The routine writes related halfwords before and after adding the incoming 32-bit value to an address component. It also stores the low half of captured S12 at twice the doubled-and-incremented result.

A 32-bit value is then read from a recently modified halfword destination and used for a byte `0x45` store at offset `+0x45`. The original halfword destination is modified again. A later 32-bit reload from it supplies a potentially different base for a 32-bit `0x00000045` store at `+0x45`.

Between those reads, the S15-derived `+0x45` location receives a halfword sum involving S13, S16, and a target-address comparison with `0x46`. It later receives a byte combining the entry-captured S01 byte, `0x53`, and the comparison using the later S12 reference.

### Linked-memory mutation region

The routine follows and rewrites 32-bit memory values. Frequent link-read offsets include `+0x04`, `+0x14`, `+0x44`, `+0x64`, and `+0x74`. Other captured word reads include `+0x50`, `+0x54`, and `+0x60`.

**The visible `+0x50` accesses are reads from byte-addressed working regions. No write at that relative offset is established.** Arbitrary aliasing could still cause another computed destination to overlap such storage.

Relative 32-bit write offsets include `+0x10`, `+0x14`, `+0x20`, `+0x24`, `+0x30`, `+0x34`, `+0x40`, `+0x44`, `+0x54`, `+0x60`, `+0x64`, and `+0x74`. Byte writes additionally use offsets including `+0x00`, `+0x01`, `+0x08`, `+0x09`, `+0x0C`, `+0x0D`, `+0x11`, `+0x15`, `+0x19`, `+0x1C`, and `+0x1D`.

These are access offsets, not a recovered object layout.

Established transitions include:

- A word recovered through `+0x14` is propagated into other regions.
- A base obtained through `+0x74` receives writes at `+0x54` and `+0x44`.
- Several destinations receive their own address as a 32-bit value.
- The absolute `0xDB` capture later replaces previously stored values at selected `+0x54` destinations.
- A byte read through a doubled address is character-promoted and right-shifted using the low byte of its address-producing link as the count. Its truncated result is propagated.
- Some locations are explicitly read and written back. Repeated writes and adjacent reloads remain represented accesses.

A region obtained through `+0x14` receives successive 32-bit values at `+0x10`: `0x73`, `0x6E`, and later a captured word read from its `+0x50` slot. It also receives 32-bit `0x6E` at `+0x20`. Its byte at `+0x1C` is written with `0x79` before `0x68`.

Later phases distinguish widths:

- Decimal `100` is written as a byte at `+0x10` and as 32-bit values elsewhere.
- `0x74` is written first as a byte and then as a 32-bit value at a common `+0x10` destination.
- `0x66` is written first as a 32-bit value and then as a byte at a common `+0x10` destination.
- `0x73` appears as a byte at `+0x0C` and later as a 32-bit value at `+0x20`.

A recovered address minus `0x74` is stored as a 32-bit value at that address minus `0x20`.

Link captures and reloads are interleaved with these writes. In particular, a central `+0x14` slot is replaced first by a traversed link and later by a value captured from another region’s `+0x64` slot. It is not a stable identity.

The complete pairwise ordering of linked-region accesses is not enumerated here and remains documented_only. This summary does not authorize batching writes, eliminating read/write-back operations, or caching links across mutations.

### Terminal effects and invocation

After S17 is captured, the routine reads a central `+0x74` link. Its low half is stored at the sum of the central base and that link. The routine also writes captured S17’s low half at twice another recovered address.

A byte is then read from a computed sum. Character promotion followed by conversion to 32-bit unsigned produces a terminal base. Depending on character signedness, bytes with the high bit set may produce either a zero-extended or sign-extended base.

Further halfword stores occur before a 32-bit `0x00000045` write at that base plus `0x45`.

S18 and S19 are then captured, followed by a 32-bit capture from the central region’s `+0x44` slot. The remaining visible writes are:

- Byte `0x44` at the terminal base plus `0x44`.
- Captured S19 as a 32-bit value at twice captured S18.
- The low half of captured S18 at a recovered value plus captured S18.

The unresolved target is invoked with:

1. The terminal base bitwise-ORed with `0x44`.
2. Captured S18.
3. The captured central `+0x44` word.

The third argument is not a reload after the final writes.

## Side effects

Writes can affect caller-reachable memory, static storage, low absolute addresses, and memory associated with dependency addresses. No output-buffer boundary is established.

Overwritten writes remain ordered effects. Their elimination would require unsupported assumptions about aliasing, faults, concurrency, and memory-mapped storage.

No ownership transfer, allocation, deallocation, lifetime extension, or reference-counting rule is established. Self-address stores do not prove ownership.

If an access fails, earlier writes may already have completed. No atomicity or rollback is established. Effects of the unresolved target are additional and unknown.

## Dependencies

| Dependency | Established relationship | Unresolved contract |
|---|---|---|
| F-32c0bc2d3e1d | Its address participates in two byte-write destinations. | No execution or preservation guarantee is established. |
| Unresolved indirect target | Address bits affect earlier calculations; invoked at the end with three represented arguments. | Identity, complete calling convention, effects, termination, return behavior, and ownership effects. |
| Static storage | Supplies interleaved captures and later references. | Initialization, overlapping layout, external mutation, and synchronization. |
| Execution environment | Supplies incoming machine state and access semantics. | Address mapping, faults, alignment, byte order, character signedness, and unresolved arithmetic behavior. |

No unknown dependency is assumed to preserve globals or to own or release memory.

## Acceptance examples

These are conditional static criteria, not observed test results.

### Entry capture before an aliased write

If the initial doubled destination overlaps S01, the later S01-derived halfword and byte use the value captured before the initial byte store. Reloading S01 afterward is not equivalent.

### Later static input observes earlier mutation

If the store through doubled S02 overlaps S03, the subsequent S03 capture occurs after that store. An implementation using an entry-time snapshot of all static inputs would not satisfy this ordering.

### Early dynamic alias

If twice the second argument overlaps the word at that argument plus `0x52`, the byte write precedes the word read. The read observes the resulting memory state under the platform’s access semantics.

### Carry-producing doubling

If captured S11 is `0x80000000` and the comparison increment is zero, its unsigned doubled low word is zero and its doubling carry is one. The companion calculation therefore includes a carry increment.

### Working-region `+0x50`

For a valid, nonaliased working region, the visible `+0x50` access captures a word for propagation. A required write to that slot cannot be inferred from this body.

### Terminal byte promotion

For a terminal byte of zero, the terminal base is zero under either character-signedness choice. The final first argument is `0x44`.

For a byte of `0x80`, zero extension yields first argument `0x000000C4`; signed promotion yields `0xFFFFFFC4`. Selecting one requires additional environment evidence.

### Captured final argument

If a final write aliases the central `+0x44` slot after its capture, the third indirect argument remains the captured word, not the slot’s updated contents.

### Shift and failure boundaries

A shift count of 32 has documented_only results until target semantics are established. A later failed access does not imply reversal of earlier successful writes.

## Uncertainties

The following remain documented_only:

- Application purpose, semantic types, and ownership.
- Complete calling convention and incoming-state requirements.
- Physical loads underlying narrowed or repeated static expressions.
- Evaluation order among references within individual expressions.
- Complete pairwise ordering of the summarized linked-memory region.
- Valid memory extents, alias configurations, and overlapping static layout.
- Character signedness, signed-shift behavior, oversized shift counts, signed overflow, and address conversion.
- Byte order, alignment, permissions, fault delivery, concurrency, and reentrancy.
- Final target identity and complete indirect-control-flow behavior.

No list, tree, allocator, string, or object invariant is imposed.

## Evidence

Established claims derive only from the supplied static representation: explicit access widths, capture boundaries, later static references, literal offsets, comparisons, carry operations, captured-value propagation, dependency-address arithmetic, and the terminal indirect invocation.

The representation reports incomplete recovery of indirect control flow. Its visible call-and-return form therefore does not establish every machine-level successor or calling detail.

Role-based storage labels and terms such as “link” and “working region” are descriptive aids, not recovered types. No runtime execution, memory trace, fault experiment, or dependency test has been performed.
