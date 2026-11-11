<!-- Trusted provenance: F-a7db53cdd9c2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7db53cdd9c2

## Purpose

F-a7db53cdd9c2 transforms externally addressed memory and prepares invocation-associated state before entering a sequence of external calls. Its observable behavior depends on access widths, retained observations, arithmetic-derived addresses, and the ordering of overlapping accesses.

No application-level data structure is established. Descriptive terms such as **link**, **successor**, and **selected object** identify address relationships, not validated objects, ownership boundaries, or collection membership.

This contract describes the supplied static representation. It does not establish that the representation is executable, that its addresses are valid, or that its terminal sequence has the behavior suggested by annotations.

## Interface

### Explicit inputs

| Argument | Represented width | Established use |
|---|---:|---|
| First | Signed 32 bits | Address base for word observations at offsets 20, 100, and 116; operand for left shifts by 4 and 8. |
| Second | Signed 32 bits | Word payload; address base for a byte destination at offset 21. |
| Third | 32 bits | No direct value use. |
| Fourth | 8 bits | Initial byte payload. |
| Fifth | 32 bits | Its invocation-associated storage address is passed to F-000000000001. |
| Sixth | 32 bits | No direct value use. |
| Seventh | 32 bits | Its invocation-associated storage is assigned zero after F-000000000001 returns. |

“No direct value use” does not establish that aliased storage is inaccessible.

Additional incoming state consists of three represented signed 32-bit values, called the **anchor**, **relay**, and **payload base**, and an indirect invocation target, F-000000000001. Three global sources provide selection values. Another global source provides an argument to F-cdec09f5d432.

Their initialization and calling convention are **documented_only**.

### Results and memory requirements

No ordinary return value is represented. The terminal sequence carries a non-return annotation; actual non-return behavior is not independently established.

All offsets below are byte offsets. A word access is 32 bits, a halfword access is 16 bits, and a byte access is 8 bits. Narrow stores preserve only the payload’s low bits. Extracting a byte from a word observation does not reduce that observation’s access width.

Displacements are observed as signed 16-bit values and extended for subsequent arithmetic. Thus, a displacement with all 16 bits set represents −1, not 65,535. Storing such a displacement as a word uses its signed 32-bit representation.

Reached accesses require memory usable for the stated width and direction. There are no represented bounds checks, address checks, alias restrictions, or recoverable error results. Signed overflow, unsupported shifts, alignment, and integer-to-address interpretation remain **documented_only**; this contract does not impose wrapping arithmetic.

## Behavior

### Observation model

The function is not an entry-snapshot transformation. A value described as **retained** belongs to its particular observation epoch. Subsequent changes to its source do not change that retained value. A **fresh** observation can reflect intervening writes.

No disjointness is established between argument-derived addresses, global sources, incoming bases, loaded addresses, arithmetic addresses, or invocation-associated storage. Access precedence therefore matters even when a write appears unrelated to a later read.

The ordering below is the ordering supported by the representation. It is not a claim about atomicity, volatile accesses, synchronization, or the number of physical instructions.

### Entry relationships

The initial memory state presented to later observations already includes a word containing the second argument at anchor offset 20 and, afterward, the fourth argument at relay offset 17.

Two retained observations then establish the **entry link** and **entry displacement**: respectively, the word at relay offset 68 and the signed halfword at relay plus anchor. Both precede the byte containing the anchor’s low bits at second-argument offset 21 and the word containing the second argument at anchor offset 68. The byte effect precedes that word effect.

The **backing address** is the later word observed at entry-link offset 116. A halfword containing the entry link’s low bits is placed at backing address plus entry displacement. The **initial carried value** is observed afterward at backing-address offset 68. An overlap between that halfword destination and the carried-value source therefore affects the carried value.

Relay offset 84 receives the relay itself as a word before selection begins.

### Selection and propagation

There are three selection phases. Each retains a selected base from its own global source and subsequently evaluates that source again for a publication word.

| Phase | Publication destination | Incoming carried value |
|---|---|---|
| First | Entry-link offset 68 | Initial carried value |
| Second | First current-link offset 68 | First outgoing carried value |
| Third | Second current-link offset 68 | Second outgoing carried value |

In every phase, selection precedes publication, and publication precedes placement of the incoming carried value at selected-base offset 20. Publication uses the later source evaluation, whereas placement uses the earlier selected base. Their equality is not guaranteed.

The first two phases additionally establish a **current link**, a signed **displacement**, and a **successor**, then propagate state.

| Relationship | First phase | Second phase |
|---|---|---|
| Working base | Relay | First successor |
| Sample | Word at first-argument offset 100 | Later word at the same address |
| Current-link source | Working-base offset 68 | Working-base offset 68 |
| Displacement source | Working base plus selected base | Working base plus selected base |
| Successor source | Displacement plus 20 | Displacement plus 20 |

The successor source depends on the signed displacement alone. It is not selected base plus displacement plus 20.

The sampling boundaries differ:

- In the first phase, working-base offset 17 receives the low byte of twice the relay before the first sample. The current-link observation follows the sample, and the displacement observation follows the current link. Successor selection follows the displacement.
- In the second phase, working-base offset 17 first receives the first sample’s low byte, before the second sample is observed. The current link and displacement are then observed in that order. Before successor selection, incoming-carried-value offset 84 receives the working base as a word, working-base offset 17 receives the second sample’s low byte, and current-link offset 68 receives the retained selected base, in that order. This additional publication does not reevaluate the global selector.

Propagation in each of these phases satisfies the following precedence constraints:

- Current-link offset 84 contains a word identifying that current link before the payload base is stored at selected base plus incoming carried value.
- That payload-base store precedes replacement of selected-base offset 20 with the current link.
- The payload word at payload-base offset 68 is observed only after this replacement. Its low byte supplies selected-base offset 13.
- The selected-base byte effect precedes a byte containing the current link at successor offset 13, which precedes a halfword containing the current link at selected base plus displacement.
- The outgoing carried value is observed at selected-base offset 68 after those propagation writes. Successor offset 84 subsequently receives the successor itself as a word.

The next phase begins after the successor self-value write. These constraints preserve both transient carried-value placement and its later replacement; specifying only final field contents would lose behavior under aliasing.

### Terminal selection and indirect address chain

The third phase publishes its selector and places its carried value but does not perform the propagation operation above.

The second successor remains the terminal working base. Its offset-17 byte receives the retained second sample’s low byte before the word at its offset 68 is retained as the **terminal link**. The signed halfword at terminal working base plus third selected base is observed afterward as the **terminal displacement**.

A repeated effect pair separates these observations from the product-derived lookup: terminal-displacement offset 84 receives the displacement as a word, followed by the terminal working base’s low byte at third-selected-base offset 17. This pair is represented twice, in that order, before the lookup.

The **selected object** is the endpoint of three dependent word observations:

| Observation | Source address |
|---|---|
| First | Terminal displacement multiplied by terminal link, plus 68 |
| Second | First observation’s value, plus 116 |
| Third | Second observation’s value, plus 20 |

The third result identifies the selected object. Neither the initial product nor either earlier observation alone identifies it.

Selected-object offset 20 supplies a retained **saved address** before that field is overwritten with twice the terminal working base. A later word observation at saved-address offset 20 supplies the operand multiplied by the terminal displacement to form the **first arithmetic address**. Because this operand is observed after the overwrite, aliases can make it reflect the replacement.

Before the word at first-arithmetic-address offset 116 is observed, terminal-displacement offset 84 again receives the displacement as a word, followed by the saved address’s low byte at third-selected-base offset 17. The offset-116 observation supplies the **terminal payload**, including its assignment to invocation-associated offset 264.

The **second arithmetic address** is the terminal displacement multiplied by the first arithmetic address. After that calculation, another displacement self-value word write precedes a terminal-payload byte write at third-selected-base offset 17.

### Final external-memory effects

The terminal path retains two word observations in order: second-arithmetic-address offset 68, then first-argument offset 116.

Third-selected-base offset 100 subsequently receives the terminal payload as a word. Its offset-9 byte then receives the first retained observation’s low byte. A fresh word observation at second-arithmetic-address offset 68 supplies the low byte stored at second-arithmetic-address offset 17. This fresh observation occurs after both selected-base writes and before its own destination write.

Another displacement self-value word write precedes a terminal-payload byte write at twice the second arithmetic address plus 17.

The last two external word observations are, in order:

- Terminal displacement multiplied by second arithmetic address, plus 68.
- First-argument offset 20, supplying the **final destination base**.

The final three external byte destinations have this precedence:

| Order | Destination | Payload |
|---|---|---|
| Earlier | Terminal payload plus 1 | Retained first-argument offset-116 value’s low byte |
| Middle | Final-destination-base offset 17 | Last product-derived word’s low byte |
| Later | Final-destination-base offset 9 | Low byte of twice the second arithmetic address |

The terminal payload therefore acts both as data and as an address base.

### Invocation-associated state and continuation

Displayed storage offsets identify represented assignments, not a proven portable structure, frame extent, or dependency-visible layout.

| Value | Displayed byte offsets |
|---|---|
| First argument shifted left 4 | 16, 36 |
| First argument shifted left 8 | 52, 76, 316, 392 |
| Terminal payload | 264, 296, 380 |
| Zero before F-000000000001 | 96, 168, 228, 440 |

Offset 264 is assigned during terminal-payload acquisition. After the final external byte effects, the remaining pre-invocation assignment order is 16, 52, 228, 96, 168, 440, 36, 76, 296, 316, 380, 392. The repeated shift-derived assignments reuse prepared values rather than establish additional evaluations of the original shifts.

F-000000000001 receives the address of the fifth argument’s storage and two zeros.

Only if it returns does the seventh argument’s storage receive zero. Displayed offset 412 then receives the 32-bit marker 0x08400E1C, followed by zero assignments at offsets 256, 288, and 320, in that order.

F-cdec09f5d432 receives the subsequently evaluated global argument value and three zeros. This argument is not an entry-time snapshot.

If that dependency returns, F-000000000002 receives zero and 0x08401154. Its represented result supplies the target of F-000000000003, an indirect invocation with no displayed arguments. The terminal sequence is annotated non-returning; its actual execution semantics remain unresolved.

## Side effects

The memory footprint is not bounded by known objects. Loaded values, signed displacements, and products become address ingredients without represented validation.

Effects are not transactionally committed. No rollback, retry, explicit allocation, explicit release, ownership transfer, or recoverable status is represented in the function’s own operations.

Passing the fifth argument’s storage address establishes neither a dependency-accessible extent nor permission to retain it. Its relationship to the displayed storage offsets and other addressable memory is unknown.

External dependencies have no established preservation guarantees. They may change globals or memory, retain pointers, allocate or release storage, or fail to return. In particular, F-000000000001 may affect the global argument observed later for F-cdec09f5d432.

Invalid memory and unsupported arithmetic have no specified fallback. Faults, traps, exceptions, and subsequent execution are **documented_only**.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-000000000001 | Incoming indirect target; fifth-argument storage address and two zeros. | Identity, ABI, memory extent, mutation, retention, ownership, preservation, and return behavior. |
| F-cdec09f5d432 | Later-observed global argument and three zeros. | Argument meaning, effects, ownership, preservation, and return behavior. |
| F-000000000002 | Terminal operation receiving zero and 0x08401154; its represented result supplies a call target. | Actual operation, continuation, and validity of a callable result. |
| F-000000000003 | Indirect invocation through that result, with no displayed arguments. | Target identity, calling requirements, effects, and termination mechanism. |

No dependency implementation is supplied. None may be assumed to preserve globals or to acquire or release ownership.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes valid accesses and supported arithmetic throughout the reached path.

1. **Access width:** A propagation source word with low byte 0xA6 supplies byte 0xA6. A byte-only source access is not equivalent to the required word observation.

2. **Entry overlap:** If the entry halfword destination overlaps backing-address offset 68, the initial carried value includes that write’s effect. An earlier snapshot is insufficient.

3. **Signed addressing:** A displacement bit pattern of 0xFFFF makes the successor source address 19. Neither 65,555 nor selected base plus 19 is the specified source.

4. **Sample lifetime:** If the first and second samples have low bytes 0x12 and 0x34, the early second-phase byte placement uses 0x12. The later second-phase replacement and third-phase placement use 0x34.

5. **Selector instability:** If a phase’s selection and publication evaluations differ, carried-value placement still uses the earlier selected base. Publication uses the later value.

6. **Publication overlap:** If the second phase’s additional publication overlaps its successor-source word, successor selection reflects that publication.

7. **Lookup depth:** With terminal displacement 2 and terminal link 256, the first lookup source is address 580. Dependent observations at offsets 116 and 20 remain necessary to identify the selected object.

8. **Overwrite-sensitive operand:** If selected-object offset 20 and saved-address offset 20 alias, the multiplication operand reflects the overwrite with twice the terminal working base. The saved address remains the earlier retained value.

9. **Different observation epochs:** If the third-selected-base writes alter second-arithmetic-address offset 68, the byte placed at selected-base offset 9 can differ from the later byte placed at arithmetic-address offset 17.

10. **Interrupted continuation:** If F-000000000001 does not return, the seventh-argument zero assignment and subsequent dependencies are not reached. If it returns after changing the later global argument source, the old value cannot be substituted on a preservation assumption.

11. **Invalid access:** The evidence requires neither successful recovery nor rollback. The actual failure mechanism is **documented_only**.

## Uncertainties

The following remain **documented_only**:

- Complete ABI, function-boundary validity, and additional incoming-state initialization.
- Endianness, alignment, address-space interpretation, and integer-to-address conversion.
- Signed overflow, unsupported left shifts, and machine-level arithmetic details not established by the representation.
- Valid memory extents, actual aliases, concurrency, and selector stability.
- Whether repeated source evaluations and identical writes correspond to separate runtime accesses.
- Physical layout and alias relationships among argument storage, displayed storage, and external memory.
- Which prepared storage values dependencies can observe.
- Dependency mutation, preservation, retention, ownership, failure, and return behavior.
- Execution semantics and actual non-return behavior of the terminal sequence.

Notices concerning omitted unreachable regions establish no reachable recovery mechanism.

## Evidence

**Established static evidence:** The supplied representation supports the stated widths, signed displacement interpretation, address dependencies, retained observation epochs, write precedence, storage assignments, and displayed invocation arguments. These are representation-level findings, not runtime observations.

**Inferred intent:** Repeated selection and state preparation may support a later dispatch operation. That interpretation is non-normative and establishes no data structure, valid address range, or ownership protocol.

**Unknowns:** Dependency implementations and target execution semantics are unavailable. Comments and annotations are not independent proof of runtime behavior.

No runtime tests, execution traces, memory captures, or independent instruction-level verification have been performed.
