<!-- Trusted provenance: F-fa370cea5c29; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa370cea5c29

## Purpose

F-fa370cea5c29 performs an ordered series of mixed-width memory accesses and finishes with an unresolved indirect invocation. The supplied static representation establishes neither an application-level purpose nor a bounded output object.

Several traversals resemble linked-record manipulation. That interpretation is **inferred intent**, not a structural or ownership guarantee. Literal values are also written into address-bearing locations; valid links cannot be assumed afterward.

This contract distinguishes represented behavior from target-specific semantics and unknown dependency behavior. All examples are conditional static acceptance criteria. No runtime tests have been performed.

## Interface

### Inputs

| Input | Represented width | Use |
|---|---:|---|
| First explicit argument | 32 bits | No visible body use. |
| Second explicit argument | Signed 32 bits | Doubled to form the first destination. |
| Third explicit argument | Signed 32 bits | Early byte-write address and working-component arithmetic. |
| Fourth explicit argument | 16 bits | Halfword payload and low-byte arithmetic. |
| Incoming word | 32 bits | Initial word payload and byte arithmetic. |
| Incoming halfword | 16 bits | First payload and later byte payload. |
| Incoming unsigned value | Unsigned 32 bits | Byte arithmetic, comparisons, and displacement adjustment. |
| Incoming character | Character width | Early byte arithmetic. |
| Indirect target | Code-address value; exact ABI unresolved | Narrow payloads, comparison, and terminal invocation. |

The incoming values other than the explicit arguments are hidden entry-state inputs. Their provenance is unknown. Fixed storage and computed memory provide additional inputs when accessed, not necessarily at entry.

### Output and completion

There is no explicit result or status value. The represented terminal invocation receives three 32-bit arguments. A return follows if that invocation returns; non-returning or exceptional target behavior remains unknown.

There is no visible validation, error reporting, rollback, allocation, or release. Completion requires every generated access to be permitted at its width and the final target to be invocable. Entry-time address validity alone is insufficient because earlier writes may change later address sources.

### Widths and notation

A byte, halfword, and word mean 8, 16, and 32 bits. All offsets are byte offsets. Unless expressly narrowed, record-field accesses below are words. Narrow stores retain the corresponding low bits.

Labels such as A, P, and “anchor” identify retained samples or address roles defined by this contract, not proven object types. Different reads remain distinct even when they reference the same storage.

Unsigned carry predicates concern 32-bit addition. Signed-overflow behavior, excessive shifts, character promotion, pointer conversion, and address comparison remain **documented_only** where the execution model is not established.

## Behavior

### Entry writes and independently sampled bases

The first write places the incoming halfword at twice the second explicit argument. A fixed-storage word is then read, doubled, and used as the destination for the incoming word.

Another fixed-storage slot supplies a retained **initial base**. The following byte destination uses the third explicit argument plus a **fresh reference to that same slot**, not necessarily the retained initial base. Its payload is the low-byte sum of the incoming word, incoming unsigned value, and incoming character.

An **early payload word** is sampled next. The incoming halfword’s low byte is written at twice the retained initial base. Only afterward is an **initial unsigned base** sampled. The initial base’s low byte is then written at twice the initial base.

Byte `0x2c` is written at initial unsigned base plus `0x2c`. A character read at twice that unsigned base supplies a promoted offset: the initial base’s low halfword is written at initial base plus that offset.

The **working component** is twice the third explicit argument, incremented by one exactly when the initial unsigned base exceeds `0x4e` under unsigned comparison. After this calculation, the early payload word’s low byte is written at twice the initial base.

The next fixed-storage slot has two separate reads:

- The first supplies the full-width **mixed-access base**.
- The second supplies a value whose low 16 bits become the **retained mixed halfword**.

The retained mixed halfword is written at initial base plus `0x53`. It is not defined as the low halfword of the first sample. Equality requires storage to remain stable across both reads.

### Mixed-width replacement and dependency address

A **shift-side base** is sampled from fixed storage. A word at working component plus mixed-access base becomes the **original mixed word**. Another fixed-storage reference supplies a retained byte payload.

A byte write uses twice a fresh reference to the shift-side slot as its destination and the low byte of another reference to that slot as its payload. These references are distinct from the retained shift-side base; their internal evaluation order is unresolved.

The fourth explicit argument is then written as a halfword at working component plus mixed-access base. The indirect target’s low halfword is written at twice the retained shift-side base.

The original mixed word, interpreted as signed 32 bits, is right-shifted by its low eight bits to obtain the **shift result**. That retained result supplies a word payload at original mixed word plus shift result. The retained shift-side base’s low halfword is then written at shift result plus shift-side base.

A **secondary base** is sampled. Subsequent effects, in order, are:

- The fourth argument’s low byte at shift result plus mixed-access base.
- Byte `0x52` at working component plus mixed-access base.
- The working component as a word at working component plus secondary base.
- The target’s low halfword at secondary base plus mixed-access base.

The address associated with F-32c0bc2d3e1d is retained as the **dependency address**. A byte at twice that address receives the retained byte payload plus the target’s low byte, truncated to eight bits. F-32c0bc2d3e1d is not invoked.

A further fixed-storage reference supplies the base for byte `0x59` at its offset `0x59`. The mixed-access base’s low byte is then written at working component plus dependency address, followed by the working component’s low halfword at working component plus mixed-access base.

### Carry-dependent displacement and comparison reload

A fixed-storage **doubling seed** is sampled. The retained mixed halfword is then written at a **fresh reference to the doubling-seed slot** plus `0x54`. This destination does not necessarily use the retained doubling seed.

An **early comparison sample** and a **sum seed** are sampled, in that order.

**Comparison bit A** is one when incoming unsigned value is at most the working component. The **doubled component** is twice the doubling seed plus comparison bit A, with the represented 32-bit unsigned arithmetic. **Carry A** is true if either doubling the seed carries, or adding comparison bit A to the truncated doubled seed carries.

The carry-source slot is referenced separately for two purposes:

1. An initial read supplies the **carry-base sample**. The **carry-adjusted base** is this sample plus carry A; it does not include `0x59`.
2. A later read supplies the **threshold sample**. If it exceeds unsigned `0xffffffa6`, **carry B** is true and the final carry-source reference is skipped.
3. Only when that threshold condition is false is the slot referenced again. **Carry B** then reports carry from adding carry A to the 32-bit result of this **conditional carry sample** plus `0x59`.

The **initial displacement** contains the mixed-access base, doubled component, and carry B. No equality among the three carry-source samples is assumed.

After initial displacement is computed, the comparison slot is reloaded. **Comparison bit B** is one when incoming unsigned value is at most this **fresh comparison sample**. The earlier comparison sample remains the source of a later halfword payload.

The initial displacement’s low halfword is written at working component plus initial displacement. Incoming unsigned value is then added to the displacement, producing the **current displacement**, whose low halfword is written at working component plus current displacement. The early comparison sample’s low halfword is written at twice the doubled component. Byte `0x45` follows at carry-adjusted base plus `0x9e`.

A **separate base** and a **halfword addend** are sampled, in that order. A word read at working component plus current displacement supplies a destination base for byte `0x45` at offset `0x45`. The current displacement’s low halfword is written again at the word-read address.

At separate base plus `0x45`, a halfword receives the low 16 bits of the sum seed’s low halfword, the halfword addend, and a bit indicating that the target compares below address value `0x46`.

The word at working component plus current displacement is reloaded as **P**. Word `0x45` is written at P plus `0x45`. Finally, the byte at separate base plus `0x45` receives the fourth argument’s low byte plus `0x53` plus comparison bit B.

The two word reads around the repeated displacement store are not interchangeable.

### Central traversal and retained records

The following paragraphs preserve access order without asserting valid structures or non-aliasing.

**Initial records.** A word **X** is read at working component plus `0x14` and written at current displacement plus `0x34`. **A** is then read from separate base plus `0x74`, followed by **B** from P plus `0x14`. P is written to A’s `0x54`, B to B’s `0x54`, and X to A’s `0x44`. A’s low byte is written to P’s `0x09`.

**Nested records.** **C** is read from A’s `0x64`, then unsigned **U** from C’s `0x14`. U’s `0x09` receives B’s low byte and then C’s low byte. A word at absolute address `0xdb` is sampled afterward as the **fixed record payload**.

**D** is read from X’s `0x14`; **extended address** retains D plus `0x20`. A word read from U’s `0x64` supplies the base for writing A at offset `0x14`. X’s `0x11` receives extended address’s low byte.

A’s `0x44` is read twice, retaining **link copy** and **link address** separately. A character at twice U is promoted, right-shifted by U’s low eight bits, and narrowed to a byte. That byte is written at link address’s `0x08`; A’s `0x74` receives link copy, and A’s `0x11` receives the derived byte.

X’s `0x54` receives extended address and then fixed record payload. X’s `0x19` receives extended address’s low byte. Link address’s `0x44` receives A, and D’s `0x74` receives extended address.

The next sequence is specifically ordered: the destination base is read from link address’s `0x14`; A’s `0x11` is overwritten with fixed record payload’s low byte; **only then** is A’s `0x14` read for the low-byte payload written at the retained destination’s `0x15`.

### Buffer and link preservation interval

A’s `0x14` is reloaded as **buffer**, followed by A’s `0x64` as **old payload**. Buffer’s `0x10` receives word `0x73`; A’s `0x44` receives A; buffer’s `0x14` receives byte `0x73`; A’s `0x54` receives fixed record payload.

A’s `0x74` supplies **early link payload**. A’s `0x09` receives A’s low byte, and buffer’s `0x54` receives early link payload. Buffer’s `0x10` then receives word `0x6e`, A’s `0x44` again receives A, and buffer’s `0x20` receives word `0x6e`.

A’s `0x64` is reloaded as **new payload**, whose low byte is written to buffer’s `0x09`. Buffer’s `0x74` supplies **H**; H’s `0x54` supplies **saved H link**.

The next byte writes remain ordered: buffer `0x1c` receives `0x79`; A `0x11` receives new payload’s low byte; A `0x09` receives old payload’s low byte; buffer `0x1c` receives `0x68`; A `0x11` again receives new payload’s low byte; H `0x11` receives A’s low byte.

Buffer’s `0x50` is sampled as **buffer payload**. A’s `0x44` receives old payload; A’s `0x64` and `0x34` receive new payload; buffer’s `0x11` receives new payload’s low byte. A’s `0x74` is then sampled as **later link payload**.

Buffer’s `0x10` receives buffer payload, H’s `0x54` receives saved H link, and buffer’s first byte receives buffer payload’s low byte. Buffer’s `0x09` receives A’s low byte.

A’s `0x14` is reloaded as **anchor**. H’s `0x11` receives buffer’s low address byte; buffer’s `0x44` receives anchor; buffer’s first byte again receives buffer payload’s low byte.

### Backward update and `0x74` interval

H’s `0x74` supplies **backward pointer**, followed by H’s `0x14` supplying **J**. Buffer’s `0x54` receives buffer itself.

J’s `0x04` supplies **K**. K’s `0x11` receives later link payload’s low byte; K’s `0x10` receives byte `100`; anchor’s `0x54` and K’s `0x60` receive word `100`.

Anchor’s `0x74` is sampled before being replaced with J. The sampled address receives word `100` at `0x30`.

J’s `0x14` supplies **L**. L’s `0x09` receives later link payload’s low byte, and anchor’s `0x54` receives anchor. A word is then written at backward pointer minus `0x20`, containing backward pointer minus `0x74`.

L’s `0x14` supplies **M**. J’s `0x11` receives early link payload’s low byte. J’s `0x14` is sampled as **J-link sample**, whose low byte is written at M’s `0x15`; J’s `0x14` is reloaded again as **N**.

Anchor’s `0x64` supplies **anchor payload**, which is immediately written back. N’s `0x0c` receives byte `0x73`, anchor’s `0x15` receives anchor payload’s low byte, and J’s `0x34` receives J-link sample.

N’s `0x74` supplies **first N link**. N’s `0x54` receives N; J’s `0x0d` receives anchor’s low byte; J’s `0x54` receives J-link sample; J’s `0x14` receives N.

Anchor’s `0x64` is sampled as **fresh anchor payload**. N’s `0x10` receives byte `0x74`, anchor’s `0x54` receives word `0x74`, and N’s `0x10` receives word `0x74`. N’s `0x74` supplies **second N link**.

J’s `0x54` receives J; N’s `0x30` receives word `0x74`; anchor’s `0x01` receives anchor payload’s low byte. The address held in anchor payload receives fresh anchor payload’s low byte at `0x09`. Anchor’s `0x74` receives fresh anchor payload, and its `0x11` receives byte `0x74`.

N’s `0x60` is sampled. Anchor’s `0x11` receives anchor payload’s low byte, anchor’s `0x34` receives fresh anchor payload, and N’s `0x10` receives the sampled word’s low byte.

### `0x66` interval and fresh traversals

Anchor’s `0x14` supplies **Q**. J’s `0x11` receives first N link’s low byte; anchor’s `0x44` receives J; anchor’s `0x11` receives fresh anchor payload’s low byte.

Anchor’s `0x74` supplies **pre-replacement link**. Anchor’s `0x0d` receives J’s low byte. Q’s `0x10` receives word `0x66`; anchor’s `0x44` again receives J; Q’s `0x10` receives byte `0x66`; anchor’s `0x54` and Q’s `0x60` receive word `0x66`.

Anchor’s `0x74` is reloaded as **R** before replacement with J. R’s `0x40` receives word `0x66`, and anchor’s `0x0d` receives pre-replacement link’s low byte.

R’s `0x44` is sampled. Anchor’s `0x54` receives anchor, and second N link’s `0x54` receives second N link itself. Next, the destination base is read from R’s `0x14`; J’s `0x11` receives the sampled R `0x44` value’s low byte; **only afterward** is J’s `0x14` read for the byte written at the retained destination’s `0x15`.

J’s `0x14` is reloaded as **next buffer**. Anchor’s `0x64` supplies **payload one**, followed by word `0x73` at next buffer’s `0x20`. Anchor’s `0x64` is sampled again as **payload two**, whose low byte is written at next buffer’s `0x09`.

Next buffer’s `0x74` supplies **T**. T’s `0x0d` receives anchor’s low byte. T’s `0x54` is read and written back. T’s `0x09` receives byte `0x74`; anchor’s `0x44` receives payload one; T’s `0x34` receives anchor.

T’s `0x04` supplies **V**. V’s `0x24` receives payload two, and anchor’s `0x19` receives payload one’s low byte.

### Final link replacement

V’s `0x74` supplies **terminal link**, followed by V’s `0x04` supplying **replacement payload**. Anchor’s `0x74` supplies **W**.

W’s `0x14` supplies the destination receiving terminal link at `0x54`. W’s `0x04` is then sampled. Anchor’s `0x0d` receives payload two’s low byte twice. Anchor’s `0x64` receives replacement payload, and the sampled W `0x04` address receives word `0x74` at `0x30`.

W’s `0x04` is reloaded; its `0x74` supplies **selected record**. Anchor’s `0x64` is reloaded, retaining its low byte for anchor’s `0x1d` and a later write.

A fresh traversal through anchor’s `0x44`, then `0x14`, supplies **replacement link**. Selected record’s `0x34` receives terminal link; anchor’s `0x14` receives replacement link. Anchor’s `0x54` supplies **saved anchor link**, and replacement link’s `0x09` receives anchor’s low byte.

A fresh traversal through selected record’s `0x04`, then `0x74`, supplies **deep record**. Deep record’s `0x15` receives terminal link’s low byte. Anchor’s `0x54` receives anchor; anchor’s `0x11` receives selected record’s low byte; deep record’s `0x11` receives saved anchor link’s low byte.

Deep record’s `0x64` supplies **deep payload**. Anchor’s `0x09` receives the byte retained from the preceding anchor `0x64` reload.

Selected record’s `0x04` is reloaded as **final buffer**. Its `0x09` receives anchor’s low byte; its `0x50` is sampled; its `0x11` receives deep payload’s low byte; its `0x09` again receives anchor’s low byte. Anchor’s `0x14` receives deep payload, anchor’s `0x0d` receives final buffer’s low address byte, and final buffer’s first byte receives the sampled `0x50` word’s low byte.

Selected record’s `0x74` supplies **terminal character base**, followed by its `0x14` supplying **final owner**. Final buffer’s `0x54` receives final buffer. Final owner’s `0x04` supplies **final object**.

Final object’s `0x10` receives byte `100`; anchor’s `0x54` and final object’s `0x40` receive word `100`. Anchor’s `0x0d` receives deep payload’s low byte; final owner’s `0x09` receives final object’s low byte; anchor’s `0x24` receives deep payload; final owner’s `0x11` receives anchor’s low byte.

### Terminal accesses and invocation

A **terminal halfword base** is sampled before anchor’s `0x74` supplies **terminal address word**.

Terminal address word’s low halfword is written at terminal address word plus anchor. Terminal halfword base’s low halfword is written at twice final object. A character at terminal character base plus anchor is promoted into the represented 32-bit **terminal character**.

Terminal address word’s low halfword is written at terminal halfword base plus terminal address word, then again at terminal address word plus anchor. Word `0x45` is written at terminal character plus `0x45`.

The final sampling order is: **final address base** from fixed storage, **final stored word** from fixed storage, then **final third argument** from anchor’s `0x44`. After all three samples, byte `0x44` is written at terminal character plus `0x44`; final stored word is written at twice final address base; final address base’s low halfword is written at terminal link plus final address base.

The indirect invocation receives, in order:

1. Terminal character bitwise ORed with `0x44`.
2. Final address base.
3. Final third argument.

Later stores do not retroactively change these retained argument values.

## Side effects

Memory mutation is the primary visible effect. Overlapping accesses, fixed-storage aliasing, executable-storage aliasing, and access-sensitive memory are not excluded.

Repeated writes, overwritten literals, and read/write-back operations remain part of the represented access trace. Matching only final memory contents does not preserve that trace where intermediate accesses are observable.

No ownership transfer follows merely from copying an address-bearing word. No allocation, release, synchronization, or rollback is visible. The indirect target’s further effects are unknown.

An access failure may leave a completed prefix of writes intact. The representation supplies no recovery mechanism requiring subsequent accesses or invocation after failure.

## Dependencies

| Dependency | Established relationship | Unknown behavior |
|---|---|---|
| F-32c0bc2d3e1d | Its address contributes to two byte-write destinations; no direct invocation. | Intended role, writable surroundings, aliasing. |
| Unresolved indirect target | Supplies narrow payloads and a comparison operand; invoked at termination. | Identity, ABI, return behavior, ownership, memory effects, global preservation. |
| Fixed storage | Supplies separately timed references described above. | Contents, overlap, external mutation, access attributes. |
| Execution environment | Determines concrete arithmetic, addressing, and access behavior. | Byte order, faults, alignment, character semantics, target-specific rules. |

No unknown dependency is assumed to preserve globals or own or release memory.

## Acceptance examples

These examples require permitted accesses and a compatible execution model; they are not executed tests.

1. **First payload:** The initial halfword write contains the hidden incoming halfword, not the fourth explicit argument.

2. **Separate mixed samples:** If the two mixed-slot reads yield `0x11223344` and `0x55667788`, the mixed-access base is `0x11223344`; the halfword payload at initial base plus `0x53`, and later at the fresh doubling-slot reference plus `0x54`, is `0x7788`.

3. **Comparison reload:** An initial unsigned base of `0x4e` contributes zero to the working component; `0x4f` contributes one. Incoming unsigned equal to the **fresh comparison sample** makes comparison bit B one, even if the early comparison sample differs. The early sample still supplies the halfword at twice the doubled component.

4. **Carry separation:** Doubling seed `0x80000000` produces carry A under unsigned 32-bit arithmetic. If the threshold sample exceeds `0xffffffa6`, carry B is one and the conditional carry-source reference is absent. The carry-adjusted base still uses the earlier carry-base sample.

5. **Conditional carry reference:** With threshold sample zero, conditional carry sample `0xffffffa6`, and carry A one, the conditional addition carries. Reusing the threshold sample would produce a different carry B and is not equivalent.

6. **Payload-read ordering:** If either intervening byte write to A’s or J’s `0x11` affects the corresponding `0x14` payload read through aliasing or access-sensitive behavior, the later payload read observes the resulting state. A payload prefetched before that byte write is not equivalent.

7. **Retained backward pointer:** The negative-offset store uses the same retained backward-pointer sample for destination-minus-`0x20` and payload-minus-`0x74`, without reloading H’s `0x74`.

8. **Transient literals:** With ordinary non-aliasing memory, buffer’s `0x1c` ends its `0x79`/`0x68` pair holding `0x68`; an access observer sees both writes.

9. **Terminal sampling:** Overlap between the terminal byte-`0x44` destination and final input storage does not change the already sampled call arguments, assuming execution continues.

## Uncertainties

The following remain **documented_only**:

- Application purpose, legitimate input domains, and completeness of the represented boundary.
- Exact fixed-storage layout and overlapping symbols.
- Byte order, alignment requirements, address validity, and fault behavior.
- Character signedness and promotion, including character-derived addresses.
- Right-shift behavior for counts from 32 through 255 and negative signed operands.
- Signed-overflowing arithmetic, address wrapping, code-address narrowing, and target-address comparison semantics.
- Evaluation order among multiple references within a single expression; ordered successive operations do not resolve that internal order.
- Concurrent, device-driven, or access-triggered storage changes.
- Whether all represented repeated accesses correspond to distinct machine accesses.
- Indirect-target identity, calling convention, preservation, ownership, and return behavior.

Stable ordinary storage can make repeated samples equal. That is an additional assumption, not permission to collapse the specified references or move reads across writes.

## Evidence

The supplied static representation supports the access widths, retained samples, separately referenced fixed storage, short-circuit carry reference, comparison reload, intervening-store ordering, traversals, negative-offset mutation, and terminal argument sampling described here.

Warnings in that representation identify overlapping fixed-storage symbols and unresolved indirect control flow. Consequently, the apparent interface and call interpretation are not independently established machine-level facts.

No runtime execution, dependency inspection, external provenance verification, or behavioral test supports this contract.
