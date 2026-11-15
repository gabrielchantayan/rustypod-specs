<!-- Trusted provenance: F-af0eb4cb6aab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af0eb4cb6aab

## Purpose

F-af0eb4cb6aab performs an ordered sequence of memory observations and publications, ending in an unresolved indirect invocation. Its behavior depends on four explicit inputs, additional incoming machine state, fixed-location state, and addresses discovered during execution.

No application-level purpose is established. Self-links, repeated offsets, and constants do not establish a container type, initialization protocol, or ownership invariant.

This contract distinguishes **established static evidence**, **inferred intent**, and unresolved behavior marked **documented_only**. No runtime tests have been performed.

## Interface

### Inputs

| Input | Established use |
|---|---|
| First explicit input | Signed 32-bit operand of the opening right shift. |
| Second explicit input | Complete 32-bit stored value; a subsequent memory reload contributes to address formation. |
| Third explicit input | 32-bit stored value, low-byte source, and address base for reads at offsets `0x04` and `0x05`. |
| Fourth explicit input | 32-bit stored value, low-byte source, and address operand. |

Additional incoming state supplies a **primary base**, **secondary base**, shift-control word, additional 32-bit data word, unsigned 32-bit arithmetic operand, byte-sized arithmetic operand, and unresolved indirect target. The target is used both numerically and as the final invocation destination.

These names describe contract roles, not recovered parameter identifiers. Their origins and the complete calling convention are documented_only.

### Widths, addresses, and result

A **word**, **halfword**, and **byte** mean 32, 16, and 8 bits respectively. Offsets are byte offsets. Pointer-like values have a represented width of 32 bits. Narrowing retains the low bits; unsigned byte loads are zero-extended. Loads explicitly described as signed bytes are sign-extended when promoted.

Fixed-state observations are word observations even where their results are immediately narrowed. No fixed cell is assumed stable across distinct references.

There is no explicit result value. If the final invocation returns normally, this routine returns without an explicit result.

Supported execution requires every actual memory access and the indirect transfer to be valid in the execution environment. Bounds, alignment, nullability, allocation, and non-aliasing guarantees are not established.

## Behavior

### Ordering model

The regions below occur successively. A **capture** retains the observed value despite subsequent memory changes. A **reread** is a distinct observation.

Duplicate stores, overlapping widths, overwritten values, and explicit read-and-write-back operations remain observable operations. References within a compound expression do not establish an otherwise unresolved evaluation order. Separate references to the same fixed cell must not be collapsed.

### Opening publications and root capture

The secondary base receives the third input at `0x74` before primary offset `0x44` supplies a related destination. The first input is shifted right using the shift-control word’s low eight bits. The result’s low byte is stored first at related-destination offset `0x08`, and later at primary offset `0x11`, after primary offset `0x74` receives the second input.

Secondary offset `0x54` receives the third input and then the fourth input as separate word stores. Secondary offset `0x19` receives the third input’s low byte. Related-destination offset `0x44` receives the primary base; absolute address `0x6e` receives its low byte.

A word read at third-input offset `0x04` supplies a destination whose offset `0x54` receives the primary base. Only afterward is third-input offset `0x05` read as an unsigned byte and used as an unscaled address.

Primary offset `0x54` receives word `0x65`, followed by word `0x65` at that byte-derived address plus `0x65`. Primary offset `0x74` is then captured and copied to primary offset `0x54`. The fourth input’s low byte is stored at byte-derived offset `0x11`.

A fixed **root pointer** is captured before primary offsets `0x34` and `0x54` receive the second and fourth inputs. Traversal uses this captured root even if those stores alter its fixed cell. The captured primary-offset-`0x74` word minus `0x70` supplies a **derived base**; this numeric use does not require executable memory.

### Captured-root traversal

The root’s unsigned byte at `0x05` is captured before its word at `0x50` is copied to `0x20`. Root offset `0x74` supplies a **link**, and link offset `0x64` supplies a **further base**. A fresh root-offset-`0x05` byte supplies root offset `0x11`; root offset zero is then captured as the **root word**.

The following publications preserve their order:

- Link `0x11` receives the root address’s low byte; root `0x09` receives the link’s low byte; root `0x54` receives the further base.
- Link `0x05` is captured as an unsigned byte address before link `0x54` receives the root word.
- Further-base `0x0c` and `0x15` receive the link byte and additional incoming word’s byte.
- Link `0x34` receives the derived base; captured-byte-address offset `0x64` receives the further base.
- Further-base `0x09` receives the additional word’s byte; byte-address offset `0x54` receives its complete word.

Further-base offset `0x60` is captured. Byte-address offset `0x11` receives the further base’s byte, followed by the captured data byte at further base plus root word.

The root-word byte is next stored at fourth input plus the initially captured root byte. This precedes both a halfword publication of the captured data at root-byte plus `0x45` and a pointer read at derived base plus further base. That pointer’s offset `0x45` receives word `0x45`; afterward, the fourth input’s byte overwrites the first byte of the earlier halfword destination.

### Fixed-state arithmetic

A fixed arithmetic word is captured. A separate observation of its cell supplies the displacement for halfword `0x53` at derived base plus that observation. The preceding pointer then receives word `0x53` at offset zero. The captured arithmetic word and target’s low halfword contribute the halfword stored at further base plus incoming unsigned operand plus `0x53`.

A **working-base cell** is separately observed as a captured base, a retained byte, and the destination operand for a doubled-address byte store. That store uses the retained byte. Derived base plus captured working base receives the low byte of further base plus incoming unsigned operand. A signed byte loaded at twice the preceding pointer, added to the captured working base, supplies a halfword destination receiving the working base’s low halfword.

The **first comparison-derived address** is twice the derived base plus the unsigned comparison bit “target is at or below derived base.”

At twice the working base, the earlier further-base-offset-`0x60` byte is stored, a fixed **saved-data word** is captured, and the retained working-base byte is stored. A word is then read at working base plus `0x52`. Subsequent halfword stores publish `0x52` at first comparison-derived address plus working base, the loaded word’s low halfword at loaded-word plus `0x52`, and the target halfword at twice working base. The loaded word is then stored at twice a separately observed fixed destination.

Another fixed base is captured. A fresh observation of its cell supplies the destination component for a byte at first comparison-derived address plus that observation. Its data combines the loaded word, incoming unsigned operand, and incoming byte operand, narrowed to eight bits.

A fixed **auxiliary word is captured before** the target’s low byte is stored at twice this captured base. A fixed unsigned **comparison word** is captured after that store and before the captured base’s low byte is stored at the same doubled address.

Comparison-word plus `0x2c` receives byte `0x2c`. A signed byte read at twice the comparison word, added to the captured base, supplies a halfword destination receiving the base’s low halfword.

The **second comparison-derived address** is twice the first plus the unsigned bit “comparison word exceeds `0x4e`.” The auxiliary word’s low byte then replaces the byte at twice the captured base.

### Self-controlled shift and surviving observations

A **retained arithmetic base** is captured from a fixed cell. A separate observation of that cell supplies the halfword stored at the preceding base plus `0x53`.

A **shift-region base** is captured next. A full word at second comparison-derived address plus retained arithmetic base is captured before any following overwrite of that location. A code-associated fixed word is then observed and narrowed to a retained byte.

The next byte store has two additional observations of the shift-region fixed cell: its destination is twice one fresh observation, and its data is the low byte of another fresh observation. Neither must equal the captured shift-region base. Their expression-internal evaluation order is unresolved.

After that byte store, the captured-word source receives the saved-data low halfword, and twice the captured shift-region base receives the target halfword.

The captured source word is interpreted as signed 32-bit and shifted right by its own low eight bits. This self-controlled shift supplies a full-word store of its result at original captured word plus shifted result. Shifted result plus captured shift-region base receives that base’s low halfword.

A further fixed destination base is captured. The saved-data byte is stored at shifted result plus retained arithmetic base; byte `0x52` overwrites the captured-word source. Second comparison-derived address plus the new destination base receives the second comparison-derived address as a word. New destination base plus retained arithmetic base receives the target halfword.

The numeric address of F-32c0bc2d3e1d participates in a doubled-address byte store combining the retained code-associated byte and target byte. Next, another separately observed fixed base plus `0x59` receives byte `0x59`. Second comparison-derived address plus the dependency address then receives the retained arithmetic base’s byte.

Finally, the captured-word source receives the second comparison-derived address’s low halfword. No invocation of F-32c0bc2d3e1d is established.

### Carry-dependent publications

A fixed unsigned **doubling operand** is captured. A separate reference to its cell supplies the base for a halfword store at `0x54`, using the earlier separately observed arithmetic-base halfword.

A **comparison-cell word** and **halfword-contributor word** are captured, in that order.

The **doubled sum** is the low 32 bits of twice the doubling operand plus the bit “incoming unsigned operand is at or below the second comparison-derived address.” The **first carry** is the logical OR of the carry from doubling and the carry from adding that bit.

One code-associated fixed observation plus the first carry supplies a later byte-store base. Additional observations determine the **second carry**: a fresh observation exceeding unsigned `0xffffffa6` makes it true; otherwise, another observation is increased by `0x59` at 32-bit width and checked for carry when the first carry is added. These observations need not agree.

Retained arithmetic base plus doubled sum plus second carry forms an arithmetic offset. Before the next four stores, the comparison cell is reread and the unsigned bit “incoming operand is at or below this reread” is retained.

The stores publish, in order:

1. The offset’s low halfword at second comparison-derived address plus offset.
2. After increasing the offset by the incoming unsigned operand, its new low halfword at the corresponding address.
3. The earlier comparison-cell capture’s low halfword at twice the doubled sum.
4. Byte `0x45` at the code-associated byte-store base plus `0x9e`.

A fixed **destination base** and a separate code-associated **halfword contribution** are captured before a pointer is read through the latest arithmetic-derived location. Pointer offset `0x45` receives byte `0x45`.

That source location receives the arithmetic offset’s low halfword before destination-base offset `0x45` receives a halfword combining the halfword-contributor capture, code-associated contribution, and unsigned bit “target is below `0x46`.”

The arithmetic-derived source is reread as a word. This later pointer receives word `0x45` at `0x45`. Destination-base offset `0x45` then receives a byte combining the saved-data byte, `0x53`, and the retained comparison-cell reread bit.

### Hub acquisition and absolute capture

A word at second comparison-derived address plus `0x14` becomes an **anchor-like value**, published at current arithmetic offset plus `0x34`. Destination-base offset `0x74` supplies the **hub**; the latest pointer’s offset `0x14` supplies another object.

Hub `0x54` receives that pointer, the other object self-links at `0x54`, and hub `0x44` receives the anchor-like value. The **hub address’s low byte is retained** and stored at pointer offset `0x09`.

Following hub `0x64`, then `0x14`, captures an unsigned word. At that word plus `0x09`, successive byte stores publish the other object’s byte and the intermediate linked base’s byte. A full word is then captured at absolute address `0xdb`.

Anchor-like offset `0x14` is captured and increased by `0x20`. Following unsigned-word offset `0x64` supplies a destination whose `0x14` receives the hub before anchor-like `0x11` receives the increased value’s byte.

Hub `0x44` is read twice. The second capture supplies the destination at `0x08` for a shift-derived byte; the first supplies hub `0x74`. The shift uses a signed byte from twice the unsigned word, promoted to signed 32-bit, and that word’s low eight bits as count. Its narrowed result also supplies hub `0x11`.

Anchor-like `0x54` receives the increased value and then the captured absolute word; `0x19` receives the increased value’s byte. The second hub-field capture’s `0x44` receives the hub. The original, pre-increase base’s `0x74` receives the increased value.

### Ordered linked rewrites

**Area preparation.** A link is acquired through the second hub-field capture’s `0x14`. Hub `0x11` receives the absolute capture’s byte. A word at hub `0x14` supplies the byte stored at link `0x15`; hub `0x14` is then reread as the **area**.

Hub `0x64` is captured before area `0x10` receives word `0x73`. Hub self-links at `0x44`; area `0x14` receives byte `0x73`; hub `0x54` receives the absolute capture.

Hub `0x74` is captured next. Hub `0x09` receives the **retained hub-address byte from acquisition**, not this field capture’s byte. The field capture supplies the complete word at area `0x54`.

Area `0x10` receives word `0x6e`, hub again self-links at `0x44`, and area `0x20` receives word `0x6e`. Hub `0x64` is captured again; its byte supplies area `0x09`.

**Area restoration and working-object acquisition.** Area `0x74` supplies a branch whose `0x54` is captured. Ordered byte stores publish area-`0x1c = 0x79`, hub-`0x11 = second hub-0x64 capture byte`, hub-`0x09 = first capture byte`, area-`0x1c = 0x68`, the same hub-`0x11` byte, and branch-`0x11 = retained hub byte`.

Area `0x50` is captured. Hub `0x44`, `0x64`, and `0x34` receive the first, second, and second captures respectively; area `0x11` receives the second capture’s byte. Hub `0x74` is captured again.

Area `0x10` receives its captured `0x50` word; branch `0x54` receives its earlier capture. Area zero and `0x09` receive the `0x50` byte and hub byte. Hub `0x14` supplies the **current anchor**. Branch `0x11` receives the area byte; area `0x44` receives the anchor; the area-zero byte store repeats.

Branch `0x74` and `0x14` are captured before area self-links at `0x54`. The latter capture is the **working object**. Its `0x04` supplies a leaf receiving the latest hub-`0x74` byte at `0x11`, byte decimal `100` at `0x10`, and word decimal `100` at `0x60`; anchor `0x54` receives word decimal `100` between the latter two leaf stores.

Anchor `0x74` is captured before replacement by the working object; the old destination receives word decimal `100` at `0x30`. Working-object `0x14` supplies another link, whose `0x09` receives the latest hub-`0x74` byte. Anchor self-links at `0x54`. Retained branch-`0x74` address minus `0x20` receives that address minus `0x74`.

**The `0x74` transition.** The last link’s `0x14` supplies a destination before working-object `0x11` receives the earlier area-preparation hub-`0x74` byte. Working-object `0x14` is captured; its byte is stored at destination `0x15`. Working-object `0x14` is then distinctly reread as the **current linked object**.

Anchor `0x64` is captured and written back before linked `0x0c` receives byte `0x73`. The anchor-field capture’s byte supplies anchor `0x15`. The earlier working-object-field capture supplies working-object `0x34`; linked `0x74` is captured before linked self-links at `0x54`.

Working-object `0x0d` receives the anchor byte. Working-object `0x54` receives the earlier working-object-field capture, but working-object `0x14` receives the **current linked object obtained by the distinct reread**.

Anchor `0x64` is captured again. Linked `0x10` receives byte `0x74`, anchor `0x54` receives word `0x74`, and linked `0x10` receives word `0x74`. Linked `0x74` is separately retained before working-object self-links at `0x54` and linked `0x30` receives word `0x74`.

The first anchor-`0x64` capture’s byte supplies anchor `0x01`; its full value supplies the base whose `0x09` receives the second capture’s byte. Anchor `0x74` receives the second capture. Anchor `0x11` receives byte `0x74`, linked `0x60` is captured, and anchor `0x11` receives the first capture’s byte. Anchor `0x34` receives the second capture; linked `0x10` receives the captured `0x60` byte.

**The `0x66` transition.** Anchor `0x14` is read before working-object `0x11` receives the earlier linked-`0x74` byte. Anchor `0x44` receives the working object and `0x11` receives the second anchor-`0x64` capture’s byte. Anchor `0x74` is captured before anchor `0x0d` receives the working-object byte.

The newly read destination receives word `0x66` at `0x10`; anchor `0x44` again receives the working object; destination `0x10` receives byte `0x66`. Anchor `0x54` and destination `0x60` receive word `0x66`.

Anchor `0x74` is reread before replacement by the working object. The reread pointer receives word `0x66` at `0x40`; anchor `0x0d` receives the earlier `0x74` capture’s byte. The reread pointer’s `0x44` is captured before anchor `0x54` and the separately retained linked-`0x74` destination’s `0x54` become self-links.

**Captures straddling word `0x73`.** The reread pointer’s `0x14` supplies a destination. Working-object `0x11` receives the captured `0x44` byte; a fresh working-object-`0x14` word supplies destination `0x15` as a byte.

Working-object `0x14` is reread as the next destination. Anchor `0x64` is captured before word `0x73` is stored at destination `0x20`, then captured again afterward. The second capture’s byte supplies destination `0x09`.

Destination `0x74` supplies a link. Link `0x0d` receives the anchor byte; link `0x54` is explicitly read and written back; link `0x09` receives byte `0x74`. Anchor `0x44` receives the first capture; link `0x34` receives the anchor.

Link `0x04` supplies a late destination. Its `0x24` receives the second capture; anchor `0x19` receives the first capture’s byte. Late-destination `0x74` is retained for the final halfword store, and `0x04` is captured separately.

**Final traversal.** Anchor `0x74` supplies a branch. Its `0x14` supplies a destination receiving the retained late value at `0x54`. Branch `0x04` is captured. Anchor `0x0d` receives the second straddled capture’s byte twice; anchor `0x64` receives the late destination’s captured `0x04` word. Captured branch-`0x04` destination `0x30` receives word `0x74`.

Branch `0x04` is reread and followed through `0x74`, yielding the **traversed branch**. Anchor `0x64` is read for its byte, stored at anchor `0x1d`. Following anchor `0x44`, then `0x14`, supplies a destination. Traversed-branch `0x34` receives the retained late value; anchor `0x14` receives that destination.

Anchor `0x54` is captured before destination `0x09` receives the anchor byte. Following traversed-branch `0x04`, then `0x74`, supplies another object. Object `0x15` receives the retained late byte; anchor self-links at `0x54`; anchor `0x11` receives the traversed-branch byte; object `0x11` receives the captured anchor-`0x54` byte.

Object `0x64` is captured before anchor `0x09` receives the earlier anchor-`0x64` byte. Traversed-branch `0x04` supplies a final area. Its `0x09` receives the anchor byte, its `0x50` is captured, its `0x11` receives the object-`0x64` byte, and its `0x09` receives the anchor byte again.

Anchor `0x14` receives the object-`0x64` word; anchor `0x0d` receives the final-area byte; final-area zero receives its captured `0x50` byte. Traversed-branch `0x74` and `0x14` are captured before final-area self-links at `0x54`.

The latter capture’s `0x04` supplies the final leaf. Leaf `0x10` receives byte decimal `100`; anchor `0x54` receives word decimal `100`; leaf `0x40` receives word decimal `100`. Anchor `0x0d`, the preceding link’s `0x09`, anchor `0x24`, and that link’s `0x11` receive, respectively, the object-`0x64` byte, leaf byte, complete object-`0x64` word, and anchor byte.

### Final stores and invocation

A fixed word is captured before anchor `0x74` supplies a **dispatch word**. The dispatch word’s low halfword is stored at dispatch word plus anchor. Twice the final leaf receives the fixed capture’s low halfword.

A signed byte is loaded at retained traversed-branch-`0x74` value plus anchor, sign-extended, then converted to unsigned 32-bit. The dispatch halfword is stored at fixed capture plus dispatch word, then again at dispatch word plus anchor. Word `0x45` is stored at signed-byte-derived value plus `0x45`.

The final sequence captures a late fixed word for argument two, captures a different fixed data word, and captures anchor `0x44` for argument three. It then stores byte `0x44` at signed-byte-derived value plus `0x44`, stores the captured data word at twice the late fixed word, and stores that late word’s low halfword at retained late-destination-`0x74` value plus late fixed word.

The unresolved target is invoked with:

1. The signed-byte-derived unsigned word bitwise ORed with `0x44`.
2. The captured late fixed word.
3. The captured anchor-offset-`0x44` word.

Final stores do not replace these captured arguments.

## Side effects

All described memory stores are part of the represented behavior. Inputs, fixed cells, linked memory, low absolute addresses, and dependency-address-derived destinations have no established separation.

No allocation, release, validation, error result, rollback, or recovery is visible before dispatch. Unsupported execution may stop after earlier stores have occurred; exact fault behavior is documented_only.

Ownership preservation is not established. Stores may discard references. The unresolved target may modify globals, retain references, allocate or release memory, fail, or not return.

## Dependencies

| Dependency or state | Established role | Unresolved aspects |
|---|---|---|
| F-32c0bc2d3e1d | Its numeric address contributes to destinations and stores. | No invocation or behavioral dependency is established. |
| Unresolved indirect target | Numeric operand and final invocation destination. | Identity, complete ABI, effects, ownership, and return behavior. |
| Fixed-location state | Supplies the distinct observations described above. | Initial contents, overlap, stability, and platform meaning. |
| Execution environment | Provides memory, arithmetic, shifts, and indirect transfer. | Faults, alignment, exceptional control flow, and unspecified arithmetic details. |

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

1. **Opening shift:** With count zero and first input `0x123456ab`, both opening shift publications contain byte `0xab`.

2. **Captured root:** A primary-offset-`0x34` store that changes the root cell does not replace the root already captured for traversal.

3. **Unscaled byte address:** A third-input-offset-`0x05` byte of `0x20` places the associated word-`0x65` store at address `0x85`.

4. **Auxiliary capture before aliasing store:** If the doubled-base target-byte store changes the auxiliary cell, subsequent use of the auxiliary byte still comes from the observation made before that store.

5. **Fresh shift-region observations:** If the captured shift-region base and the later destination and data observations differ, the first byte destination uses twice the destination observation and its data uses the data observation’s low byte. No ordering between those two expression-internal observations is required.

6. **Distinct working-object observations:** If the destination-`0x15` byte store changes working-object `0x14`, working-object `0x34` and `0x54` receive the earlier capture, while `0x14` receives the distinct reread value.

7. **Hub-address byte:** If the hub address’s low byte differs from the captured hub-`0x74` word’s low byte, area preparation stores the former at hub `0x09` and the complete latter word at area `0x54`.

8. **Carry boundary:** Doubling operand `0x80000000` with comparison bit zero produces doubled sum zero and first carry one. If the relevant code-associated observations equal `0xffffffa6`, the threshold comparison is false, but adding `0x59` and then the first carry produces the second carry.

9. **Early comparison:** A carry-region halfword store that changes the comparison cell does not change the comparison bit retained before those stores.

10. **Straddled captures:** If word `0x73` changes anchor `0x64`, the first capture still supplies anchor `0x44` and its `0x19` byte; the second supplies the destination-`0x09` byte and late-destination-`0x24` word.

11. **Final signed byte:** Loaded byte `0x80` becomes unsigned word `0xffffff80`; the first invocation argument is `0xffffffc4`.

12. **Captured argument:** A final store aliasing anchor `0x44` does not replace the third argument captured beforehand.

## Uncertainties

The following remain **documented_only**:

- Routine boundaries, recovered control flow, implicit-input origins, and complete calling convention.
- Valid addresses, allocations, object extents, structure types, endianness, alignment, and fault behavior.
- Large-count semantics for the three distinct signed right shifts.
- Signed overflow and interpretation of surrounding address arithmetic.
- Fixed-state stability, overlap, and unresolved expression-internal observation order.
- Writability of low absolute and dependency-address-derived destinations.
- Target identity, additional invocation state, ownership effects, and return behavior.

Explicit unsigned carry calculations establish 32-bit carry behavior where represented. They do not establish portable wrapping for every surrounding signed expression.

Linked-state manipulation is inferred intent only. No list, tree, reference-counting, or other application invariant is established.

## Evidence

The supplied static representation establishes the described access widths, sequential capture boundaries, distinct rereads, narrowing, sign extension, comparisons, carry dependencies, repeated stores, and final indirect invocation.

Comments, prior prose, and reviewer feedback are not independent behavioral evidence. The four disputed observation boundaries are supported by the supplied sequence: auxiliary capture precedes the target-byte store; working-object `0x14` receives the distinct reread; the first shift-region byte store contains separate fresh destination and data observations; and the area-preparation hub-`0x09` publication uses the retained hub-address byte.

No implementation execution, runtime observation, or tests were performed.
