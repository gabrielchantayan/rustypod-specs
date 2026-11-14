<!-- Trusted provenance: F-ecf2a16a12a6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ecf2a16a12a6

## Purpose

F-ecf2a16a12a6 performs an ordered sequence of memory reads, byte/halfword/word writes, pointer-dependent mutations, and a terminal indirect invocation.

No application-level purpose is established. Repeated constants do not establish text processing, and pointer-shaped values do not establish valid objects or a particular container operation. Linked-state manipulation is inferred intent only.

This contract describes the supplied static representation, not independently validated executable behavior. Successful completion requires every reached address and the indirect target to be usable under the target’s arithmetic and access rules.

## Interface

### Explicit inputs

| Input | Represented width | Use |
|---|---:|---|
| First address component | Signed 32 bits | Added to the second component for the first halfword destination. |
| Second address component | Signed 32 bits | Contributes to that destination. |
| Primary base | Unsigned 32 bits | Address base, stored data, and unsigned-comparison operand. |
| Payload | 16 bits | Halfword data and low-byte contributions. |

Signed representations do not establish portable behavior for overflowing address calculations.

### Ambient inputs and fixed storage

Ambient inputs include a 16-bit machine-state value, an unsigned 32-bit machine-state value, an unresolved indirect target, and the represented address of F-32c0bc2d3e1d.

The following neutral slot labels identify fixed storage referenced by the representation. They do not imply independent locations or immutable values.

| Slot | Principal uses |
|---|---|
| 1 | Initial offset and separately obtained initial halfword. |
| 2 | Early doubled destinations and narrow data. |
| 3 | Captured byte used in dependency-address arithmetic. |
| 4 | Primary-relative word destination and ambient-halfword destination. |
| 5 | Destination component for byte `0x59`. |
| 6 | Carry arithmetic and an early halfword destination. |
| 7 | Captured halfword data and a separately evaluated comparison. |
| 8 | Contribution to a halfword sum. |
| 9 | Carry-adjusted address and overflow predicate. |
| 10 | Halfword-sum destination base and initial traversal base. |
| 11 | Captured halfword contribution to that sum. |
| 12 | Captured word used during early linked-state mutation. |
| 13 | Late halfword data and destination component. |
| 14 | Final doubled destination, halfword data, and second indirect argument. |
| 15 | Final word data. |

The supplied representation warns that some fixed symbols overlap smaller symbols. Non-aliasing must not be assumed.

### Output

No return value is represented. Outputs are memory mutations and one terminal indirect invocation with three represented 32-bit arguments:

1. The final character-derived unsigned word, bitwise ORed with `0x44`.
2. The captured slot-14 value.
3. A word captured from working-object offset `0x44`.

If the indirect dependency returns normally, execution reaches the represented return. There is no separate success status.

## Behavior

### Access and arithmetic model

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. Represented pointer accesses are 32 bits. Unless explicitly called a byte or halfword, fields below are words. Field offsets are hexadecimal byte offsets.

Narrow stores retain the low 8 or 16 bits. Endianness determines their effect on overlapping words. Captured values retain their captured contents even if the source is subsequently overwritten.

Ordering below follows successive represented operations. It does not establish operand evaluation order within a single expression. Fresh references must not be replaced with earlier captures merely because they name the same storage.

Unsigned 32-bit arithmetic supports modulo-\(2^{32}\) results and carry detection. Signed overflow, conversion of out-of-range unsigned values to signed values, signed shifts, and pointer arithmetic require target-specific rules. Descriptions of wrapped addresses are conditional on a 32-bit bit-pattern interpretation.

Object names below identify captured addresses only. They do not establish allocation boundaries, distinctness, ownership, or valid types.

### Initial captures and narrow writes

Slot 1 first supplies a word-sized **initial offset**, then separately supplies an **initial halfword**. The initial halfword is written at the sum of the first two explicit inputs.

Slot 2 is then captured. A word is read at primary base plus initial offset, and slot 3’s low byte is captured.

Next, in order:

- A byte containing slot 2’s low byte is written at twice slot 2, using fresh slot-2 references rather than necessarily the captured value.
- The payload halfword is written at primary base plus initial offset.
- The ambient halfword is written at twice the captured slot-2 value.

The previously read primary-relative word is interpreted as signed 32-bit data and shifted arithmetically right by its own low byte. The shift result is written as a word at the original captured word plus that result. The captured slot-2 low halfword is written at shift result plus captured slot 2.

Slot 4 is captured. The following writes occur in order:

1. Payload low byte at shift result plus initial offset.
2. Byte `0x52` at primary base plus initial offset.
3. Primary base as a word at primary base plus captured slot 4.
4. Ambient halfword at captured slot 4 plus initial offset.

The address of F-32c0bc2d3e1d is then used as data, not invoked. At twice that address, a byte is written containing the low-byte sum of captured slot 3 and the indirect target’s represented value.

Next come byte `0x59` at slot 5 plus `0x59`, initial-offset low byte at primary base plus the dependency address, and primary-base low halfword at primary base plus initial offset.

### Carry-dependent offsets and repeated pointer reads

Slot 6 is captured. The initial halfword is written at a destination formed from a separate slot-6 reference plus `0x54`. Slots 7 and 8 are then captured.

An unsigned comparison contributes one if the ambient unsigned word is less than or equal to primary base, otherwise zero. This bit is added to twice captured slot 6. A first carry records overflow from either doubling or adding that bit.

Slot 9 plus the first carry forms an address component. Separately evaluated slot-9 references determine a second bit using:

- An unsigned comparison against `0xffffffa6`; or
- If necessary, carry from adding the first carry to slot 9 plus `0x59`.

With stable slot-9 contents, this second bit expresses overflow from adding `0x59` and the first carry. Separate references remain significant if contents can change.

The first derived offset is initial offset plus the doubled-and-adjusted slot-6 value plus the second bit. Before the following writes, the ambient unsigned word is compared with a fresh slot-7 reference; this comparison is independent of the slot-7 capture.

In order:

1. The first derived offset’s low halfword is written at primary base plus that offset.
2. The ambient unsigned word is added to the offset, producing the **updated offset**; its low halfword is written at primary base plus updated offset.
3. Captured slot-7 low halfword is written at twice the doubled-and-adjusted slot-6 value.
4. Byte `0x45` is written at the carry-adjusted slot-9 component plus `0x9e`.

Slot 10 is captured, then slot 11’s low halfword. A pointer is read at primary base plus updated offset, and byte `0x45` is written at that pointer plus `0x45`. The updated-offset halfword is written again at the pointer source.

A halfword sum is then written at captured slot 10 plus `0x45`. It combines captured slot-8 and slot-11 low halves, plus one if the indirect target’s represented address is below `0x46`.

Only afterward is the pointer source read again. Word `0x45` is written at this **second pointer** plus `0x45`. Finally, the byte at captured slot 10 plus `0x45` is replaced with the low-byte sum of payload, `0x53`, and the fresh slot-7 comparison bit.

The halfword-sum write can alter the second pointer read through aliasing.

### Initial linked-state mutation

A **parent** is read from primary-base offset `0x14`, then written at updated offset plus `0x34`, without adding primary base.

A **root** is read from captured slot-10 offset `0x74`; a **peer** is read from second-pointer offset `0x14`. In order, root `0x54` receives the second pointer, peer `0x54` receives its own address, root `0x44` receives parent, and second-pointer byte `0x09` receives root’s low byte.

Root `0x64` supplies a **carrier**; carrier `0x14` supplies an unsigned **node address**. Node byte `0x09` receives peer’s low byte, then carrier’s low byte.

Slot 12 is captured. Parent `0x14` supplies a **child**; child plus `0x20` is captured as a **derived address**. A fresh node-`0x64` read supplies a destination whose `0x14` receives root. Parent byte `0x11` receives derived-address low byte.

Root `0x44` is read twice, first as captured data and then as a destination base. A character read at twice node address is widened and arithmetically shifted right by node-address low byte. The resulting low byte is written at the second captured destination’s `0x08`. Root `0x74` then receives the first captured word, followed by root byte `0x11` receiving the shift-result byte.

Parent `0x54` receives derived address, then captured slot 12. Parent byte `0x19` receives derived-address low byte. The second captured destination’s `0x44` receives root; child `0x74` receives derived address.

The second captured destination’s `0x14` supplies another destination. Root byte `0x11` receives slot-12 low byte. A fresh root-`0x14` read supplies the low byte written at that destination’s `0x15`.

### First literal group and working-object selection

Fresh reads capture root `0x14` as the **first leaf** and root `0x64` as the **earlier root-`0x64` capture**.

In order:

- First-leaf `0x10` receives word `0x73`.
- Root `0x44` receives root.
- First-leaf byte `0x14` receives `0x73`.
- Root `0x54` receives captured slot 12.
- Root `0x74` is captured as the **earlier root-`0x74` capture**.
- Root byte `0x09` receives root’s low byte.
- First-leaf `0x54` receives the earlier root-`0x74` capture.
- First-leaf `0x10` receives word `0x6e`.
- Root `0x44` receives root again.
- First-leaf `0x20` receives word `0x6e`.

A fresh root-`0x64` capture supplies first-leaf byte `0x09`. First-leaf `0x74` supplies a **leaf link**, whose `0x54` is captured.

First-leaf byte `0x1c` receives `0x79`; root byte `0x11` receives fresh root-`0x64` low byte; root byte `0x09` receives earlier root-`0x64` low byte; first-leaf byte `0x1c` receives `0x68`; root byte `0x11` receives fresh root-`0x64` low byte again; leaf-link byte `0x11` receives root’s low byte.

First-leaf `0x50` is captured. The next assignments are explicitly:

- Root `0x44` receives the earlier root-`0x64` capture.
- Root `0x64` receives the fresh root-`0x64` capture.
- Root `0x34` receives that same fresh capture.

There is no arithmetic doubling in these assignments.

First-leaf byte `0x11` receives fresh root-`0x64` low byte. Root `0x74` is captured again as the **later root-`0x74` capture**. First-leaf `0x10` receives captured first-leaf `0x50`; leaf-link `0x54` receives its previously captured value; first-leaf byte zero receives first-leaf-`0x50` low byte; first-leaf byte `0x09` receives root’s low byte.

A fresh root-`0x14` read establishes the **working object**. Leaf-link byte `0x11` receives first-leaf low byte; first-leaf `0x44` receives working object; first-leaf byte zero receives captured first-leaf-`0x50` low byte again.

### Decimal-100 group and working-`0x64` reread

Leaf-link `0x74` and `0x14` are captured separately; the latter is the **branch**. First-leaf `0x54` receives its own address.

Branch `0x04` supplies an object whose byte `0x11` receives later root-`0x74` low byte. Its byte `0x10` receives decimal `100`, working `0x54` receives word `100`, and its `0x60` receives word `100`.

Working `0x74` is read, then replaced with branch. Its old target receives word `100` at `0x30`. Branch `0x14` supplies an object whose byte `0x09` receives later root-`0x74` low byte. Working `0x54` receives its own address.

At captured leaf-link-`0x74` minus `0x20`, a word containing that captured value minus `0x74` is written.

The branch-linked object’s `0x14` supplies a byte destination. Branch byte `0x11` receives earlier root-`0x74` low byte. A fresh branch-`0x14` word supplies the low byte written at destination `0x15`. Another branch-`0x14` read establishes the **second leaf**.

Working `0x64` is read and immediately written back. Second-leaf byte `0x0c` receives `0x73`; working byte `0x15` receives that working-`0x64` capture’s low byte; branch `0x34` receives the captured branch-`0x14` word.

Second-leaf `0x74` is captured. Second-leaf `0x54` receives its own address; branch byte `0x0d` receives working-address low byte; branch `0x54` receives captured branch-`0x14`; branch `0x14` receives second leaf.

Only then is working `0x64` reread. This value need not equal the earlier read-and-write-back value.

### `0x74` and `0x66` groups

Second-leaf byte `0x10` receives `0x74`; working `0x54` receives word `0x74`; second-leaf `0x10` receives word `0x74`. Second-leaf `0x74` is captured again; branch `0x54` receives its own address; second-leaf `0x30` receives word `0x74`.

Working byte `0x01` receives the first working-`0x64` capture’s low byte. At that captured address plus `0x09`, the reread value’s low byte is written. Working `0x74` receives the reread word; working byte `0x11` receives `0x74`.

Second-leaf `0x60` is captured. Working byte `0x11` receives first working-`0x64` low byte; working `0x34` receives the reread word; second-leaf byte `0x10` receives captured second-leaf-`0x60` low byte.

Working `0x14` supplies a **third leaf**. Branch byte `0x11` receives the earlier second-leaf-`0x74` capture’s low byte. Working `0x44` receives branch; working byte `0x11` receives reread working-`0x64` low byte. Working `0x74` is captured, then working byte `0x0d` receives branch low byte.

Third-leaf `0x10` receives word `0x66`; working `0x44` receives branch again; third-leaf byte `0x10` receives `0x66`; working `0x54` and third-leaf `0x60` receive word `0x66`.

Working `0x74` is reread and replaced with branch. Its old target receives word `0x66` at `0x40`. Working byte `0x0d` receives the preceding working-`0x74` capture’s low byte.

The old target’s `0x44` is captured. Working `0x54` receives its own address, and the most recently captured second-leaf-`0x74` target receives its own address at `0x54`. The old target’s `0x14` supplies a destination. Branch byte `0x11` receives captured old-target-`0x44` low byte; destination byte `0x15` then receives a fresh branch-`0x14` word’s low byte.

### Late relinking

A fresh branch-`0x14` read establishes another leaf. Working `0x64` is captured; that leaf’s `0x20` receives word `0x73`; working `0x64` is captured again. The second capture’s low byte is written at leaf `0x09`.

Leaf `0x74` supplies a **late link**. Late-link byte `0x0d` receives working-address low byte; late-link `0x54` is read and written back; late-link byte `0x09` receives `0x74`. Working `0x44` receives the first recent working-`0x64` capture; late-link `0x34` receives working address.

Late-link `0x04` supplies a **subordinate**. Subordinate `0x24` receives the second recent capture. Working byte `0x19` receives the first capture’s low byte.

Subordinate `0x74` supplies a **terminal link**; subordinate `0x04` supplies another captured word. A fresh working-`0x74` read supplies a **current branch**. Current-branch `0x14` supplies an object whose `0x54` receives terminal link. Current-branch `0x04` is captured.

Working byte `0x0d` receives the second recent working-`0x64` low byte twice consecutively. Working `0x64` receives captured subordinate-`0x04`; captured current-branch-`0x04` target receives word `0x74` at `0x30`.

Current-branch `0x04` is reread and followed through `0x74`, establishing a **late anchor**. Working `0x64` is reread; its low byte is written at working `0x1d`.

A fresh working-`0x44` read is followed through `0x14`. Late-anchor `0x34` receives terminal link, then working `0x14` receives the newly reached address.

Working `0x54` is captured. The newly reached object’s byte `0x09` receives working-address low byte. Late-anchor `0x04`, followed through `0x74`, establishes a **late destination**. Late-destination byte `0x15` receives terminal-link low byte.

Working `0x54` receives its own address; working byte `0x11` receives late-anchor low byte; late-destination byte `0x11` receives captured working-`0x54` low byte. Late-destination `0x64` is captured, then working byte `0x09` receives the latest working-`0x64` low byte.

A fresh late-anchor-`0x04` read establishes the **final leaf**. Final-leaf byte `0x09` receives working-address low byte; final-leaf `0x50` is captured; final-leaf byte `0x11` receives late-destination-`0x64` low byte; final-leaf byte `0x09` receives working-address low byte again.

Working `0x14` now receives captured late-destination `0x64`, a second, distinct replacement. Working byte `0x0d` receives final-leaf low byte; final-leaf byte zero receives captured final-leaf-`0x50` low byte.

Late-anchor `0x74` and `0x14` are read separately. Final-leaf `0x54` receives its own address. The captured `0x14` target’s `0x04` supplies the **final reached object**.

Final-reached-object byte `0x10` receives decimal `100`; working `0x54` receives word `100`; final-reached-object `0x40` receives word `100`. Working byte `0x0d` receives late-destination-`0x64` low byte. The captured `0x14` target’s byte `0x09` receives final-reached-object low byte; working `0x24` receives late-destination `0x64`; that target’s byte `0x11` receives working-address low byte.

### Final accesses and invocation

Slot 13 is captured, then working `0x74`. Captured working-`0x74` low halfword is written at captured working `0x74` plus working address.

Slot-13 low halfword is written at twice final reached object. A character is read at captured late-anchor `0x74` plus working address and converted to an unsigned 32-bit value.

Captured working-`0x74` low halfword is written at slot-13 capture plus captured working `0x74`, then again at working address plus captured working `0x74`. Word `0x45` is written at the character-derived value plus `0x45`.

Slots 14 and 15 are captured, followed by working `0x44`. Byte `0x44` is written at character-derived value plus `0x44`; captured slot-15 word is written at twice captured slot 14; slot-14 low halfword is written at terminal link plus captured slot 14.

The indirect dependency is then invoked with the arguments specified under Interface.

## Side effects

Destinations include caller-derived, fixed-derived, pointer-derived, doubled, and character-derived addresses. Any write may affect a later source, pointer field, or fixed slot unless external constraints exclude aliasing.

Repeated writes, overwritten writes, and read-and-write-back operations remain part of the represented access sequence.

No allocation, release, reference acquisition, or ownership transfer is established in the body. Self-links do not imply ownership. The indirect dependency’s retention, release, and global-preservation behavior are unknown.

There is no represented validation, rollback, synchronization, or explicit error path. An invalid access can prevent completion after earlier mutations have occurred; fault reporting and recovery are unspecified.

## Dependencies

| Dependency | Established use | Unknown contract |
|---|---|---|
| F-32c0bc2d3e1d | Address used as arithmetic data and in destination formation; no invocation established. | Executable behavior is irrelevant to the represented body. |
| Unresolved indirect dependency; opaque ID unavailable | Represented address used in byte arithmetic and comparison; invoked once at the end. | Identity, complete calling convention, effects, ownership, exceptions, and return behavior. |
| Ambient state and fixed storage | Supplies values at the stated capture points. | Initialization, aliasing, storage attributes, and preservation requirements. |

No unknown dependency may be assumed to preserve globals or acquire or release memory.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Separate initial reads:** If slot 1 supplies `0x12345678` for the offset and `0xabcdef01` for the initial halfword, the offset remains `0x12345678` and the first halfword is `0xef01`.

- **Ordinary shift:** A captured primary-relative word of `0x00000101` gives count one and result `0x00000080` under ordinary signed 32-bit shift semantics. Its word destination is `0x00000181`, if accessible.

- **Distinct pointer reads:** If the halfword-sum write changes the primary-relative pointer source, the second pointer read observes the resulting contents. Reusing the first pointer is not equivalent.

- **Root captures are copied, not doubled:** If earlier root `0x64` was captured as `0x12345678` and fresh root `0x64` as `0x01020304`, the specified stores supply `0x12345678` to root `0x44`, then `0x01020304` to root `0x64`, then `0x01020304` to root `0x34`. Neither latter store supplies `0x02040608`. Later aliasing may still overwrite these destinations.

- **Working-`0x64` reread:** If an intervening byte or branch-field write aliases working `0x64`, its reread can differ from the earlier read-and-write-back value.

- **Two working-`0x14` replacements:** The address reached through working `0x44` is stored first; captured late-destination `0x64` is stored later. Reporting only the final contents omits an established write.

- **Repeated byte stores:** Both consecutive writes to working byte `0x0d` remain represented operations despite equal supplied values.

- **Character signedness:** For a final character byte `0xff`, signed-character interpretation produces unsigned word `0xffffffff`, unchanged by OR with `0x44`. Unsigned-character interpretation produces `0x000000ff`. Associated address calculations remain subject to target rules.

- **Normal completion:** If all accesses succeed and the indirect dependency returns normally, the routine reaches its represented return without a success code.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the supplied boundary and recovered indirect-control-flow representation.
- Endianness, alignment, address mapping, and access-fault behavior.
- Character signedness at both character reads.
- Shift behavior for counts greater than 31.
- Signed overflow, unsigned-to-signed conversion, and address-arithmetic rules.
- Operand evaluation order within expressions containing multiple references.
- Ambient-state initialization and complete calling convention.
- Aliasing, asynchronous mutation, volatile storage, and device-memory semantics.
- Object types, allocation extents, lifetime, and ownership.
- Identity and full behavior of the indirect dependency.

## Evidence

Evidence consists solely of the supplied static representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

The representation establishes access widths, successive operation ordering, separate capture points, constants, truncations, carry predicates, pointer dependencies, and the terminal invocation.

In particular, the three root-field stores after the first-leaf-`0x50` capture copy the earlier root-`0x64` value once and the fresh root-`0x64` value to two separate fields. They contain no doubling operation.

Application intent, data-structure semantics, and executable equivalence remain unestablished. No runtime tests, memory traces, architecture checks, or independent executable validation have been performed.
