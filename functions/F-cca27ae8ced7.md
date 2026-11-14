<!-- Trusted provenance: F-cca27ae8ced7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cca27ae8ced7

## Purpose

F-cca27ae8ced7 mutates an externally supplied memory graph through an order-sensitive sequence of byte, halfword, and word accesses, then invokes an unresolved indirect dependency.

The supplied static representation establishes neither an application-level purpose nor a validated object model. Repeated offsets and self-address stores suggest linked structures, but insertion, deletion, initialization, string construction, and resource release are not established behaviors.

This contract describes the represented access trace. It does not establish that the recovered entry boundary, implicit inputs, or indirect-call interpretation match executable behavior. No runtime tests have been performed.

## Interface

### Explicit and implicit inputs

| Input | Represented width | Use |
|---|---:|---|
| First argument | 32 bits | No visible use. |
| Second argument | 32 bits | Initial displacement arithmetic and a doubled-address halfword store. |
| Third argument | 32 bits | Initial address base and later word access at offset `0x14`. |
| Fourth argument | 8 bits | Early byte arithmetic. |
| Incoming unsigned value | 32 bits | Carry calculation and byte-store address. |
| Incoming additive value | 32 bits | Initial displacement arithmetic. |
| Incoming comparison value | 32 bits | Unsigned comparison and displacement addition. |
| Incoming carry | Represented as 8 bits | Carry calculation and address addition; its flag interpretation ordinarily requires zero or one. |
| Indirect destination | Address-like value | Early comparison and final invocation. |

These implicit-input descriptions do not assert a calling convention.

The represented interface has no return value. A return follows the indirect invocation in the supplied representation; whether the destination returns is unknown.

### Memory and arithmetic conventions

All offsets below are byte offsets. Unless explicitly identified as byte or halfword, a field access is a 32-bit word access. Byte and halfword writes retain the low eight and sixteen bits respectively. “Low byte” and “low halfword” describe numeric truncation, not a particular byte order.

Names such as A, P, and E identify captured address values, not distinct allocations. Snapshot names identify values captured at particular reads. They remain unchanged by later memory writes, although later reads of the same field may differ.

Successful completion requires every accessed address to support its actual width and direction. No visible null, bounds, alignment, permission, or graph-integrity checks establish those conditions.

Address overflow, signed arithmetic, pointer ordering, byte order, and alignment behavior require target information not supplied here. The carry calculation specifically uses 32-bit addition. The byte-shift operation and final byte-to-word conversion have additional signedness uncertainties described below.

## Behavior

### Trace requirements

The following groups form one continuous ordered trace. Operations within each paragraph occur in the stated order. Group boundaries do not permit reordering.

Every described read is a separate observation unless explicitly called a saved value. Aliasing is unrestricted by the representation. Consequently, intervening stores may change later pointer reads, including reads from apparently unrelated records or external storage.

External storage is identified by role rather than private symbol or address.

### 1. Entry arithmetic and overlapping writes

The function first reads an external comparison word, saving it for an early halfword store, and then an external arithmetic word.

The initial displacement combines the incoming additive value, the second argument, and a carry-derived increment. The increment is one if the incoming unsigned value exceeds `0xFFFFFFA6`, or if adding the incoming carry to the 32-bit result of adding `0x59` produces carry-out. Otherwise it is zero.

The comparison storage is read again for an unsigned less-than-or-equal comparison with the incoming comparison value. This second observation supplies a saved Boolean; the first observation supplies the later halfword store. They must not be conflated.

The initial displacement’s low halfword is stored at the third argument plus that displacement. The incoming comparison value is then added to the displacement. This updated displacement is used throughout the remaining entry operations.

Its low halfword is stored at the third argument plus the updated displacement. The first external comparison observation’s low halfword is stored at twice the second argument. Byte `0x45` is stored at the incoming unsigned value plus incoming carry plus `0x9E`.

An external base address is captured, followed by an external value narrowed to a halfword. A word is read at the third argument plus the updated displacement; byte `0x45` is stored at that loaded address plus `0x45`. The displacement halfword is then stored again at the word-read location.

The external base’s halfword at `0x45` receives the low sixteen bits of the sum of the earlier arithmetic word, the captured external halfword, and one if the indirect destination compares below address `0x46`.

The displacement-based word is read again, establishing **A**. A’s word at `0x45` receives `0x45`. The external base’s byte at `0x45` then receives the fourth argument plus `0x53` plus the saved comparison Boolean, truncated to eight bits.

### 2. Establishing traversal roles

The third argument’s word at `0x14` is captured as **B**. B is immediately stored at the updated displacement plus `0x34`, without adding the third argument. Only afterward are **C**, from the external base’s `0x74`, and **A-child**, from A’s `0x14`, captured.

C’s `0x54` receives A; A-child’s `0x54` receives A-child; C’s `0x44` receives B; A’s byte `9` receives C’s low byte.

C’s `0x64` is read as an intermediate address, whose `0x14` is read as **D**. D’s byte `9` receives A-child’s low byte, then the intermediate address’s low byte.

An external word, called the **link word**, is captured. B’s `0x14` is captured as **B-child**, and B-child plus `0x20` becomes the **advanced address**.

D’s `0x64` is read; that target’s `0x14` receives C. B’s byte `0x11` receives the advanced address’s low byte. C’s `0x44` is then read twice: first as a saved word, then as the destination address **H**.

A byte is read at twice D. The representation promotes it with signed-character behavior, shifts it right by D’s low eight bits, and narrows the result to a byte. That byte is stored at H’s `8`. C’s `0x74` receives the first C-`0x44` observation, and C’s byte `0x11` receives the shifted byte.

### 3. Early rewrites and P establishment

B’s `0x54` receives the advanced address, then the link word. B’s byte `0x19` receives the advanced address’s low byte. H’s `0x44` receives C, and B-child’s `0x74` receives the advanced address.

H’s `0x14` is captured as a byte-destination base **J** before C’s byte `0x11` receives the link word’s low byte. C’s `0x14` is then read, and its low byte is stored at J’s `0x15`.

A fresh C-`0x14` read establishes **P**. C’s `0x64` is captured as **C64-early**. The next mutations and capture occur in this exact interleaving:

1. P’s word `0x10` receives `0x73`.
2. C’s `0x44` receives C.
3. P’s byte `0x14` receives `0x73`.
4. C’s `0x54` receives the link word.
5. C’s `0x74` is captured as **C74-early**.
6. C’s byte `9` receives C’s low byte.
7. P’s `0x54` receives C74-early.
8. P’s word `0x10` receives `0x6E`.
9. C’s `0x44` receives C again.
10. P’s word `0x20` receives `0x6E`.

### 4. P-associated snapshots and self-links

C’s `0x64` is read again as **C64-later**. P’s byte `9` receives its low byte. P’s `0x74` establishes **Q**, whose `0x54` is saved as **Q54**.

P’s byte `0x1C` receives `0x79`; C’s byte `0x11` receives C64-later’s low byte; C’s byte `9` receives C64-early’s low byte; P’s byte `0x1C` receives `0x68`; C’s byte `0x11` again receives C64-later’s low byte. Q’s byte `0x11` receives C’s low byte.

P’s `0x50` is captured as **P50**. C’s `0x44` receives C64-early; C’s `0x64` and then `0x34` receive C64-later; P’s byte `0x11` receives C64-later’s low byte.

C’s `0x74` is captured as **C74-later**. P’s `0x10` receives P50; Q’s `0x54` receives Q54; P’s byte zero receives P50’s low byte; P’s byte `9` receives C’s low byte.

A fresh C-`0x14` read establishes **E**. Q’s byte `0x11` receives P’s low byte; P’s `0x44` receives E; P’s byte zero again receives P50’s low byte.

Q’s `0x74` is captured as the **back address**, followed by Q’s `0x14` as **parent**. P’s `0x54` then receives P. Only after that store is parent’s offset `4` read as **constant target**.

### 5. First constant group and repeated parent traversals

Constant target’s byte `0x11` receives C74-later’s low byte; its byte `0x10` receives decimal `100`; E’s word `0x54` receives `100`; constant target’s word `0x60` receives `100`.

E’s `0x74` is captured as **old target**. E’s `0x74` is replaced with parent before old target’s word `0x30` receives `100`.

Parent’s `0x14` is captured as **first child**. Its byte `9` receives C74-later’s low byte. E’s `0x54` receives E. At back address minus `0x20`, a word store writes back address minus `0x74`.

First child’s `0x14` is captured as **byte target**. Parent’s byte `0x11` receives C74-early’s low byte. A fresh parent-`0x14` read is saved as **parent-child snapshot**; its low byte is stored at byte target’s `0x15`. Another parent-`0x14` read establishes **T**.

### 6. The `0x73` and `0x74` mutations

E’s `0x64` is captured as **E64-first**, then written back. T’s byte `0x0C` receives `0x73`; E’s byte `0x15` receives E64-first’s low byte; parent’s `0x34` receives parent-child snapshot.

T’s `0x74` is captured as **T74-first**. T’s `0x54` receives T. Parent’s byte `0x0D` receives E’s low byte; parent’s `0x54` receives parent-child snapshot; parent’s `0x14` receives T.

E’s `0x64` is captured again as **E64-second**. T’s byte `0x10` receives `0x74`; E’s word `0x54` receives `0x74`; T’s word `0x10` receives `0x74`. T’s `0x74` is read again as **T74-second**.

Parent’s `0x54` receives parent. T’s word `0x30` receives `0x74`. E’s byte `1` receives E64-first’s low byte. At E64-first plus `9`, a byte store writes E64-second’s low byte.

E’s `0x74` receives E64-second; E’s byte `0x11` receives `0x74`. T’s `0x60` is captured as **T60**. E’s byte `0x11` receives E64-first’s low byte; E’s `0x34` receives E64-second; T’s byte `0x10` receives T60’s low byte.

### 7. The `0x66` mutations

E’s `0x14` is captured as **next child**. Parent’s byte `0x11` receives T74-first’s low byte. E’s `0x44` receives parent; E’s byte `0x11` receives E64-second’s low byte.

E’s `0x74` is saved as **saved E74**. E’s byte `0x0D` receives parent’s low byte; next child’s word `0x10` receives `0x66`; E’s `0x44` receives parent again; next child’s byte `0x10` receives `0x66`; E’s word `0x54` receives `0x66`; next child’s word `0x60` receives `0x66`.

E’s `0x74` is read again as **former target**, then replaced with parent. Former target’s word `0x40` receives `0x66`. E’s byte `0x0D` receives saved E74’s low byte. Former target’s `0x44` is saved as **former link**.

E’s `0x54` receives E. T74-second’s `0x54` receives T74-second. Former target’s `0x14` is captured as **later byte target**. Parent’s byte `0x11` receives former link’s low byte. A fresh parent-`0x14` read supplies the low byte stored at later byte target’s `0x15`.

### 8. Later traversals and E replacement

Another parent-`0x14` read establishes **current child**. E’s `0x64` is captured as **E64-third**. Current child’s word `0x20` receives `0x73`. A fresh E-`0x64` read establishes **E64-fourth**, whose low byte is stored at current child’s byte `9`.

Current child’s `0x74` establishes **linked record**. Its byte `0x0D` receives E’s low byte. Its `0x54` is read and written back. Its byte `9` receives `0x74`; E’s `0x44` receives E64-third; linked record’s `0x34` receives E.

Linked record’s offset `4` establishes **branch**. Branch’s `0x24` receives E64-fourth. E’s byte `0x19` receives E64-third’s low byte. Branch’s `0x74` is captured as the **retained traversal address**, followed by branch’s offset `4` as **branch-four**.

E’s `0x74` establishes **current parent**. Current parent’s `0x14` is read, and that target’s `0x54` receives the retained traversal address. Current parent’s offset `4` establishes **first-four**.

E’s byte `0x0D` receives E64-fourth’s low byte twice consecutively. E’s `0x64` receives branch-four. First-four’s word `0x30` receives `0x74`.

Current parent’s offset `4` is read again; this fresh target’s `0x74` establishes **traversal record**. E’s `0x64` is read again, saving its low byte as **saved E64 byte**. E’s byte `0x1D` receives that byte.

E’s `0x44` is read, then that target’s `0x14` is read as the **replacement**. Traversal record’s `0x34` receives the retained traversal address before E’s `0x14` receives replacement.

E’s `0x54` is saved as **saved E54**. Replacement’s byte `9` receives E’s low byte. Traversal record’s offset `4` is read, followed by that target’s `0x74`, establishing **surface record**.

Surface record’s byte `0x15` receives the retained traversal address’s low byte. E’s `0x54` receives E; E’s byte `0x11` receives traversal record’s low byte; surface record’s byte `0x11` receives saved E54’s low byte.

### 9. Final graph mutations

Surface record’s `0x64` is captured as **surface64**. E’s byte `9` receives saved E64 byte immediately before traversal record’s offset `4` is read as **U**.

U’s byte `9` receives E’s low byte; U’s `0x50` is captured as **U50**; U’s byte `0x11` receives surface64’s low byte; U’s byte `9` receives E’s low byte again.

E’s `0x14` receives surface64; E’s byte `0x0D` receives U’s low byte; U’s byte zero receives U50’s low byte.

Traversal record’s `0x74` is captured as **final back address**, then its `0x14` as **V-parent**. U’s `0x54` receives U. V-parent’s offset `4` establishes **V**.

V’s byte `0x10` receives decimal `100`; E’s word `0x54` receives `100`; V’s word `0x40` receives `100`. E’s byte `0x0D` receives surface64’s low byte; V-parent’s byte `9` receives V’s low byte; E’s `0x24` receives surface64; V-parent’s byte `0x11` receives E’s low byte.

### 10. Dispatch preparation

An external address-bearing word is captured as **external address one**. E’s `0x74` is captured as **final E74**.

Final E74’s low halfword is stored at final E74 plus E. External address one’s low halfword is stored at twice V.

A byte is read at final back address plus E. The represented conversion sign-extends this byte before conversion to unsigned 32 bits, producing the **dispatch address**.

Final E74’s low halfword is stored at external address one plus final E74, then again at final E74 plus E. A word `0x45` is stored at dispatch address plus `0x45`.

A second external address-bearing word, **external address two**, is read, followed by an external data word and E’s `0x44`, saved as **final link**.

The final stores are, in order:

- Byte `0x44` at dispatch address plus `0x44`.
- The external data word at twice external address two.
- External address two’s low halfword at retained traversal address plus external address two.

F-indirect-01 is invoked with three represented arguments: dispatch address bitwise OR `0x44`, external address two, and final link.

## Side effects

The visible body performs extensive overlapping memory mutation, repeated stores, self-address stores, and read/write-backs. No explicit allocation, deallocation, synchronization, validation, rollback, or error-reporting operation is visible.

A word write affects four bytes, even when its value fits in one byte. A later byte write does not generally replace the complete word.

An invalid access may prevent subsequent operations after earlier stores have occurred. No transactional or defined recovery behavior is established. Memory ownership and lifetime are not established, either for the body or for the indirect dependency.

## Dependencies

| Opaque ID | Established role | Unresolved contract |
|---|---|---|
| F-indirect-01 | Its destination value participates in the early comparison; it receives the final three arguments. | Identity, ABI, return behavior, register/global preservation, memory effects, ownership, and release behavior are documented_only. |

External storage is independently observable at its stated read positions. It must not be treated as an immutable entry snapshot.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

- **Unused argument:** Changing only the first argument does not change the represented body’s accesses when all other explicit inputs, implicit state, and memory observations remain identical.
- **Comparison reread:** If the two comparison-storage reads differ, the first supplies the doubled-second-argument halfword store; the second determines the Boolean.
- **Comparison boundary:** Equality at the second comparison read contributes one. An unsigned greater-than comparison contributes zero.
- **Byte truncation:** Fourth argument `0xFF` with comparison contribution one produces early byte `0x53`.
- **Carry boundary:** Incoming unsigned value `0xFFFFFFA7` produces increment one. At `0xFFFFFFA6`, carry zero produces zero and carry one produces one.
- **Aliasing during establishment:** If the displacement-plus-`0x34` store changes the external base’s `0x74`, C must reflect the changed value.
- **Captured traversal:** If C’s byte-`0x11` overwrite changes H’s `0x14` through aliasing, J remains the value loaded before that overwrite.
- **Post-store traversal:** If P’s self-address store changes parent’s offset `4`, constant target must reflect the subsequent read.
- **Distinct T snapshots:** T74-first supplies the later parent byte `0x11`; T74-second identifies the later self-address destination. Equal values are not required.
- **Write trace:** P’s byte `0x1C` receives `0x79` before `0x68`; final-state equality alone is insufficient.
- **Signed dispatch byte:** Under the represented signed-byte interpretation, byte `0x80` becomes `0xFFFFFF80`, making the first invocation argument `0xFFFFFFC4`.
- **Late failure:** No defined error return or undo of preceding writes is established when a late address is inaccessible.

## Uncertainties

The following are **documented_only**:

- Authentic entry boundary, implicit-state initialization, and calling convention.
- Accuracy of recovered control flow and the interpretation of the final indirect transfer as a returning call.
- Byte order, alignment rules, address mapping, device-memory semantics, and concurrent modification.
- Signed-character interpretation, signed right-shift behavior, and shift counts outside the target’s supported range.
- Arithmetic overflow and address wrapping beyond the explicit 32-bit carry calculation.
- Physical access width of the external value represented only through halfword narrowing.
- Pointer-order comparison semantics for the indirect destination.
- External-storage stability, graph validity, ownership, and lifetimes.
- All unspecified effects of F-indirect-01.

Linked-structure intent remains inference only. It imposes no additional acceptance conditions.

## Evidence

Established evidence consists exclusively of the supplied static representation: represented input widths, implicit-state references, arithmetic dependencies, access widths, access ordering, truncations, repeated reads, and indirect invocation arguments.

Warnings concerning overlapping external symbols and unresolved indirect control flow limit interpretation. No runtime execution, binary validation, trace capture, or test result supports stronger claims.
