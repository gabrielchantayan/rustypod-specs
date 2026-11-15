<!-- Trusted provenance: F-cdddb87f224d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdddb87f224d

## Purpose

F-cdddb87f224d performs an ordered sequence of address-dependent memory mutations, prepares invocation-associated state, and invokes an entry-selected dependency. If that dependency returns, it prepares further state and invokes F-cdec09f5d432. If that call returns, execution reaches an operation annotated as non-returning.

No application-level purpose is established. Repeated offsets and address-valued fields do not establish a list, allocator, object format, ownership protocol, or persistent structural invariant.

This contract describes static evidence, conditional on execution reaching each operation. Application-level intent remains unknown. Unresolved matters are marked **documented_only**. No runtime tests have been performed.

## Interface

### Displayed arguments and additional entry state

| Argument | Represented width | Established use |
|---|---:|---|
| 1 | 32 bits | Supplies the base of an offset-0x64 word read. |
| 2 | 32 bits | Incoming value not explicitly consumed. |
| 3 | 32 bits | Incoming value not explicitly consumed. |
| 4 | 8 bits | Supplies an early byte write. |
| 5 | 32 bits | Associated storage is exposed by address to the indirect dependency. |
| 6 | 32 bits | Incoming value not explicitly consumed. |
| 7 | 32 bits | Assigned zero after the indirect dependency returns. |

The displayed arguments do not establish the complete entry boundary or calling convention. Assignment to argument 7 does not establish a caller-visible output parameter. The extent and lifetime of argument-5-associated storage are **documented_only**.

Additional consumed entry state supplies an **opening destination**, **companion address**, **initial working base**, **entry word payload**, and **indirect call target**. These are descriptive roles, not guaranteed distinct allocations.

Three byte writes consume values without preceding definitions in the supplied body. They are **unresolved byte payloads 1–3**. Later assignments do not establish their earlier values.

No ordinary return value is established.

### Widths, memory, and preconditions

Offsets are byte offsets. A **word** is 32 bits, a **halfword** is 16 bits, and a **byte** is 8 bits. The displacement halfwords are signed. Narrow stores retain the low bits of their specified width. Address-valued arithmetic operands are represented as 32-bit quantities; the complete ABI remains unknown.

Field names describe accesses, not allocation layouts. Computed destinations may overlap other fields, global sources, arguments, or invocation-associated storage.

Every reached access requires appropriately readable or writable storage for its width. Reached calls require compatible targets and calling conventions. The body does not validate these preconditions.

## Behavior

### Ordering and terminology

The represented path before the calls is straight-line. Reads observe memory at their positions in that order. A captured value remains the value used by later operations even if its source is subsequently overwritten. A fresh read may observe intervening writes.

There are 22 distinct global selection sources, **G01–G22**, numbered by their represented consumption order. **G23** supplies the later direct call. Distinct source locations need not contain distinct values or designate distinct allocations.

Unless a stage specifies a departure, its **handoff** consists of:

- Capturing the outgoing selection’s offset-0x44 word.
- Writing the current working base’s own address to its offset 0x54.
- Obtaining the next global selection and installing that source’s value in the designated predecessor’s offset 0x44.
- Writing the captured outgoing word to the new selection’s offset 0x14.

Selection and installation separately mention the global source in the representation. Neither a physical load count nor an atomic snapshot is established.

A handoff defines word transfers only. Each halfword store’s destination and payload are specified separately below. A **doubled-base byte** means the low eight bits of twice the then-current working base, written at that base’s offset 0x11.

### Opening and G01–G04

The opening reads a signed halfword at initial working base plus companion address. Its low byte is written to opening-destination offset 0x15. Companion offset 0x0D then receives the companion address’s low byte. A halfword at companion plus the captured signed displacement receives the opening destination’s low sixteen address bits.

Companion offset 0x44 is captured after those writes. The initial base receives its self-address word at offset 0x54. G01 is installed in opening-destination offset 0x44, and the G01 selection’s offset 0x14 receives the companion capture.

Argument 4 is written to initial-base offset 0x11 before the word read at argument 1 plus 0x64.

**G01.** Initial-base offset 0x44 supplies a node before the signed-halfword read at initial base plus selection. The node receives its own address at offset 0x34 and the companion capture’s low byte at offset 0x0D. Initial-base offset 0x0D receives the node’s low byte. The halfword at selection plus signed displacement receives the node’s low sixteen bits. The node is the predecessor for the handoff to G02. Initial-base offset 0x11 then receives the argument-1-relative capture’s low byte.

**G02.** A fresh initial-base offset-0x44 node read precedes the base-plus-selection signed-halfword read. The node receives a self-address word at offset 0x34 and the transferred word’s low byte at offset 0x0D. Initial-base offset 0x14 then supplies a **retained address**. After the node’s offset-0x54 self-store, the halfword at selection plus displacement receives that node’s low sixteen bits. The node is the predecessor for G03; a doubled-base byte follows.

**G03.** Retained-address offset 0x64 is captured before fresh node and signed-halfword reads. Selection plus the preceding transferred word receives that word’s low byte. A fresh initial-base offset-0x64 word supplies the byte written at node offset 0x0D.

The halfword destination is three times the signed displacement, without an added selection or base. Its payload is the low sixteen bits of the node captured through initial-base offset 0x44. A word read at twice the displacement plus 0x44 replaces the normal outgoing transfer capture for the handoff to G04. The earlier retained-address payload supplies the subsequent initial-base offset-0x11 byte.

**G04.** Fresh initial-base node and signed-halfword reads precede a byte write at selection plus transferred word, containing that word’s low byte. A fresh full-word read from initial-base offset 0x64 supplies both the node-offset-0x0D byte and a later address base. Between those effects, initial-base offset 0x14 refreshes the retained address, and the node receives its offset-0x54 self-store.

The halfword destination is the full offset-0x64 capture plus signed displacement; its payload is the earlier initial-base-selected node’s low sixteen bits. Offset 0x44 is then read through that full-word capture, not through the original G04 selection. That word is transferred to G05 using the node as predecessor. A doubled-base byte follows.

### G05–G10: retained state and payload propagation

**G05.** Retained-address offset 0x64, initial-base offset 0x44, and the base-plus-selection signed halfword are captured in order. Retained-address offset 0x15 receives the incoming transferred word’s low byte. The node is rebound through its offset 0x14.

Selection offset 0x24 receives the entry word payload before selection offset 0x74 is captured. Selection offset 0x0D receives the retained-address payload’s low byte. The halfword at selection plus displacement receives the rebound node’s low sixteen bits. That node is the predecessor for G06. The retained-address payload also supplies the following initial-base offset-0x11 byte.

**G06.** Fresh node and signed-halfword reads precede another retained-address offset-0x15 write containing the incoming transfer’s low byte. The node is rebound through its offset 0x14. Selection offset 0x24 receives the saved G05 offset-0x74 word before selection offset 0x74 is captured anew.

Selection offset 0x14 is replaced by the retained address; initial-base offset 0x11 receives that address’s low byte; rebound-node offset 0x09 receives the selection’s low byte. The halfword at selection plus displacement receives the rebound node’s low sixteen bits. The handoff to G07 uses that node, followed by a doubled-base byte.

**G07.** The signed-halfword read precedes the incoming transfer’s low-byte write to retained-address offset 0x09. Initial-base offset 0x14 supplies the predecessor. A fresh retained-address offset-0x64 word supplies an initial-base offset-0x11 byte. The halfword at selection plus displacement receives the predecessor’s low sixteen bits, specifically those of the value read through initial-base offset 0x14. The handoff to G08 and a doubled-base byte follow.

**G08.** Retained-address offset 0x64 is captured before fresh node and signed-halfword reads. Retained-address offset 0x54 receives the saved G06 offset-0x74 word. Selection offset 0x09 receives the initial base’s low byte; node offset 0x0D receives the selection’s low byte. The halfword at selection plus displacement receives the initial-base-selected node’s low sixteen bits. That node is the predecessor for G09. The saved offset-0x64 payload supplies the following base-offset-0x11 byte.

**G09.** Fresh node and signed-halfword reads precede three byte writes: the selection’s own low byte at selection offset 0x01, the saved G06 payload’s low byte at selection offset 0x0D, and the selection’s low byte at node offset 0x09. The halfword at selection plus displacement receives the initial-base-selected node’s low sixteen bits. The handoff to G10 uses that node, followed by a doubled-base byte.

**G10.** The signed displacement controls a word read at displacement plus 0x44, producing an **auxiliary address**. Selection offset 0x14 supplies the predecessor. A fresh retained-address offset-0x64 word supplies a base-offset-0x11 byte. The halfword at selection plus displacement receives the predecessor’s low sixteen bits, not the auxiliary address’s bits. The selection-offset-0x44 capture becomes a **retained link** during the handoff to G11. A doubled-base byte follows.

### G11–G14: anchor and working-base changes

**G11.** Auxiliary-address offset 0x64 is captured before fresh initial-base node and signed-halfword reads. Retained-link offset 0x54 receives the initial base. Initial-base offset 0x11 receives unresolved byte payload 1, then node offset 0x44 is explicitly replaced by the selection.

A word at displacement plus 0x44 refreshes the auxiliary address. Selection offset 0x14 supplies the predecessor. The earlier auxiliary payload supplies another base-offset-0x11 byte. The halfword at selection plus displacement receives this predecessor’s low sixteen bits. The handoff to G12 uses the predecessor, followed by a doubled-base byte.

**G12.** The signed halfword from base plus selection controls a word read at displacement plus 0x44, establishing the **anchor address**. Selection offset 0x14 supplies the predecessor. A fresh auxiliary-offset-0x64 read supplies a base-offset-0x11 byte.

Anchor offset 0x04 supplies a word payload, and selection offset 0x74 supplies a retained address. Base offset 0x54 receives the payload before a halfword at twice the base plus selection receives the predecessor’s low sixteen bits. The outgoing selection-offset-0x44 capture and base self-store precede the handoff to G13. The anchor-offset-0x04 payload supplies the following base-offset-0x11 byte.

**G13.** Anchor offset 0x64, initial-base offset 0x44, and the signed halfword are captured before anchor offset 0x54 receives the retained G12 address. Selection offset 0x14 supplies a **new working base**; its offset 0x0D receives the selection’s low byte.

The halfword at selection plus displacement receives the earlier initial-base-selected node’s low sixteen bits. After the outgoing offset-0x44 capture, the new base receives the self-store. The earlier node is the predecessor for G14. New-base offset 0x11 receives the saved anchor payload’s low byte.

**G14.** The current base’s offset-0x44 node and base-plus-selection signed halfword are captured before anchor offset 0x54 again receives the retained G12 address. Selection offset 0x14 rebinds the working base, whose offset 0x0D receives the selection’s low byte.

The incoming transferred link’s offset 0x14 supplies a separate address. Retained-G12-address offset 0x54 receives the anchor. The halfword at separate address plus displacement receives the low sixteen bits of the node captured through the working base before rebinding. Separate-address offset 0x44 supplies the outgoing transfer. The rebound base receives its self-store before the handoff to G15 and its doubled-base byte afterward; the earlier node is the predecessor.

### G15–G19: byte-selected addresses and further rebinding

**G15.** A signed halfword from working base plus selection controls a byte read at displacement plus incoming transferred link. Selection offset 0x1D receives the retained G12 address’s low byte. The halfword at selection plus displacement receives the widened byte value’s low sixteen bits. The widened byte also acts as predecessor address for G16. A doubled-base byte follows.

**G16.** Another signed-halfword and link-relative byte are captured. A fresh anchor-offset-0x64 word is written at signed displacement plus 0x54. Byte-selected offset 0x0D receives the incoming link’s low byte. Selection offset 0x64 then supplies a **retained word-address**. The halfword at selection plus displacement receives the widened byte’s low sixteen bits. The byte-selected predecessor is used for G17, followed by a doubled-base byte.

**G17.** Anchor offset 0x64 is captured before the signed-halfword and link-relative byte reads. Ordered writes place the selection’s low byte at anchor offset 0x01, the incoming link’s low byte at selection offset 0x0D, the retained word-address at selection offset 0x44, and the selection’s low byte at byte-selected offset 0x0D.

The halfword at selection plus displacement receives the widened byte’s low sixteen bits. Selection offset 0x44 is freshly read afterward. The byte-selected predecessor is used for G18. The saved anchor payload supplies the following base-offset-0x11 byte.

**G18.** Fresh anchor payload, working-base offset-0x44 node, and signed-halfword reads precede an offset-0x74 read through the incoming transferred link. Working-base offset 0x14 then rebinds the base. The earlier node’s offset 0x0D receives the offset-0x74 capture’s low byte.

The halfword at that full capture plus displacement receives the earlier working-base-selected node’s low sixteen bits. The capture’s offset-0x44 word supplies the outgoing transfer. G19 uses the earlier node as predecessor and the rebound base for the self-store and following anchor-payload byte.

**G19.** Fresh node and signed-halfword reads precede a byte write of the selection’s low byte to incoming-link offset 0x15. Selection offset 0x44 is replaced by that link. Node offset 0x74 supplies an address. Its displacement-relative halfword receives the node’s low sixteen bits; its offset-0x44 word supplies the transfer to G20. The node remains predecessor. The base self-store and doubled-base byte occur in their normal positions.

### G20–G22: computed word stores and products

**G20.** Anchor offset 0x64, working-base offset 0x44, and the signed halfword are captured. A word at displacement plus 0x14 rebinds the working base. The selected node receives its offset-0x54 self-store.

A word at selection plus incoming transferred link receives the retained word-address. Selection offset 0x14 receives the node. A fresh retained-word-address offset-0x44 word supplies selection’s offset-0x0D byte; rebound-base offset 0x0D receives the node’s low byte. The halfword at selection plus displacement receives the selected node’s low sixteen bits. The outgoing selection-offset-0x44 capture, rebound-base self-store, handoff to G21 using the node, and saved anchor byte follow.

**G21.** Another anchor payload, node, and signed halfword are captured. Incoming-link offset 0x54 receives the current base; current-base offset 0x11 receives unresolved byte payload 2; node offset 0x44 is replaced by the selection.

A word at displacement plus 0x14 rebinds the base. The node self-store, computed word store, selection-offset-0x14 replacement, fresh retained-word-address read, and two offset-0x0D writes follow as in G20. The halfword at selection plus displacement receives this stage’s selected node’s low sixteen bits. The handoff to G22 uses that node and rebound base. The saved anchor payload supplies the subsequent base-offset-0x11 byte.

**G22.** Working-base offset 0x44 supplies a node, and the signed halfword at base plus selection supplies a factor. A word self-store at factor plus 0x54 alternates with a byte write of the base’s low bits at selection offset 0x11; this pair occurs twice.

Nested word reads follow factor times node plus 0x44, the resulting address’s offset 0x74, and the next address’s offset 0x14. The resulting address’s offset-0x14 word is captured before that field is replaced by twice the working base. The captured address’s offset-0x14 word is multiplied by the factor, forming an **intermediate product**.

Another factor-relative self-store and a selection-offset-0x11 byte containing the captured address’s low bits precede the intermediate product’s offset-0x74 capture. Multiplying the intermediate product by the factor yields the **current product**. A further factor-relative self-store and unresolved byte payload 3 at selection offset 0x11 precede the late effects:

- Current-product offset 0x44 and then anchor offset 0x74 are captured.
- Selection offset 0x64 receives the intermediate-product offset-0x74 capture.
- Selection offset 0x09 receives the earlier current-product offset-0x44 capture’s low byte.
- A fresh current-product offset-0x44 read supplies a byte at current-product offset 0x11.
- After another factor-relative self-store, twice current product plus 0x11 receives the intermediate capture’s low byte.
- A word at factor times current product plus 0x44 is captured, followed by an anchor-offset-0x14 destination read.
- Intermediate capture plus 0x01 receives the anchor-offset-0x74 payload’s low byte.
- Destination offset 0x11 receives the last product-derived word’s low byte; destination offset 0x09 receives doubled current product’s low byte.

The two current-product offset-0x44 reads are separate and may differ under aliasing.

### Arithmetic and calls

Signed halfwords range from −32,768 to 32,767. Tripling that range, or doubling it and adding 0x44, fits signed 32-bit arithmetic. This does not establish address validity.

Plain-byte signedness is **documented_only**: 0xFF may widen to −1 or 255, affecting byte-selected addresses and halfword payloads. Endianness, general overflow behavior, and signed-left-shift semantics are unresolved. Universal modular arithmetic is not established.

Before the indirect call, 32-bit invocation-associated locations hold anchor shifted left four bits, anchor shifted left eight bits, the intermediate-product offset-0x74 capture, and zeros. Counting original holders and copies, these occupy respectively two, four, three, and four represented locations. The four zero assignments precede the additional copies. No contiguous dependency-visible layout is established.

F-indirect-unresolved receives the argument-5-associated address and two zeros. Its result is not explicitly consumed.

Only after its return are argument 7 assigned zero, another invocation-associated location assigned 0x08400E1C, and three further locations assigned zero. G23 is obtained afterward and supplies F-cdec09f5d432’s first argument, followed by three zeros.

If that call returns, F-terminal-unresolved receives represented operands zero and 0x08401154. It is annotated as non-returning. A subsequent invocation through its represented result exists, but a usable callable result and supported continuation are not established.

## Side effects

Reached stores modify entry-derived, global-selected, computed, and invocation-associated storage. Intermediate writes, repeated stores, replacements, and fresh reads remain significant under possible aliasing.

No explicit allocation or release is identifiable in this body. Dependency memory management is unknown. Address-valued writes do not establish ownership transfer. Exposing argument-associated storage does not establish retention or release permission.

No null, bounds, alignment, overflow, or target-validity checks are represented. Earlier effects may remain if a later operation fails. No rollback, atomicity, recovery protocol, or error result is established. Fault delivery is **documented_only**.

## Dependencies

| Opaque identity | Established interface and order | documented_only |
|---|---|---|
| F-indirect-unresolved | Entry-selected target; argument-5-associated address and two zeros; return gates later preparation. | Identity, ABI, accessible storage extent, memory effects, global preservation, ownership, retention, failures, return behavior. |
| F-cdec09f5d432 | Subsequently obtained G23 value and three zeros, after indirect-call return and post-call assignments. | Argument meanings, result, memory effects, preservation, ownership, failures, return behavior. |
| F-terminal-unresolved | Zero and 0x08401154 after direct-call return; annotated non-returning. | Platform action, exceptions, resumability, result meaning, continuation. |

G01–G23 are data dependencies, not established function dependencies.

## Acceptance examples

These are conditional obligations, not executed tests.

- **Signed displacement:** Opening halfword pattern 0xFFFF denotes −1. Its companion-relative destination is one byte before the companion.
- **Early overlap:** If the argument-4 byte write overlaps the subsequent argument-1-relative word read, that read observes the preceding mutation.
- **Halfword truncation:** If G03’s selected node is 0x12345678, its halfword payload is 0x5678 at three times the signed displacement. The payload is not the displacement.
- **G04 full-word base:** If its fresh base-offset-0x64 capture is address A, A controls both the displacement-relative halfword destination and the later offset-0x44 read. The halfword payload still comes from the selected node.
- **Rebound-node payload:** If G06’s initial node and its offset-0x14 value differ, the halfword contains the latter value’s low sixteen bits.
- **Predecessor payloads:** G07 uses its initial-base-offset-0x14 predecessor; G10 and G11 use their selection-offset-0x14 predecessors. An auxiliary address is not interchangeable with those payloads.
- **Capture before rebinding:** If G14 or G18 captures node N and then rebinds its working base, the halfword payload remains N’s low sixteen bits.
- **G20/G21 selection:** Their halfword payloads come from their captured nodes, not from the retained word-address or rebound base.
- **Byte widening:** A byte pattern 0xFF leaves byte-selected addresses and widened halfword payloads conditional on unresolved signedness.
- **Fresh replacement read:** G17’s outgoing offset-0x44 read occurs after replacement and intervening writes; caching the earlier value is not generally equivalent.
- **Captured link:** G22 follows the old offset-0x14 value captured before replacement, not the newly stored doubled-base value.
- **Dependency mutation:** If the indirect dependency changes G23 and returns, the later source access observes its then-current value, subject to intervening effects.
- **Return gating:** A non-returning indirect call prevents post-call assignments and later calls. A non-returning F-cdec09f5d432 prevents the terminal operation.

## Uncertainties

The following remain **documented_only**:

- Complete entry boundary, argument placement, and additional entry-state provenance.
- Unresolved byte payloads 1–3.
- Global initialization, physical load realization, overlap, and application meanings.
- Allocation extents, permissions, alignment, endianness, byte signedness, overflow, and signed-shift semantics.
- Invocation-associated storage layout and caller or dependency visibility.
- Dependency effects, preservation, ownership, retention, failures, and returns.
- Terminal platform behavior and continuation.
- Behavior outside the supplied representation, including regions described as omitted.

These unknowns do not erase established access widths, payload dependencies, or ordering. No application-level interpretation is promoted from inferred intent to established behavior.

## Evidence

Established claims derive from the supplied representation’s access widths, signed-halfword reads, global-source occurrences, address dependencies, narrow-store payloads, captured values, replacements, repeated effects, invocation-associated assignments, and call ordering.

In particular, the halfword payload dependencies are explicit: G03–G04, G08–G09, G13–G14, and G18–G21 use their previously selected nodes; G05–G06 use rebound nodes; G07 and G10–G12 use their specified predecessors; G15–G17 use widened bytes. Destinations are independently determined as described above.

The dependency mapping establishes F-cdec09f5d432’s opaque identity, not its behavior. Prior prose and reviewer feedback are not independent behavioral evidence. Source annotations establish representation limits or attributed properties, not runtime proof.

No implementation execution, dependency inspection, runtime trace, or runtime test was performed.
