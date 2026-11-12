<!-- Trusted provenance: F-32b6e24abe5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-32b6e24abe5c

## Purpose

F-32b6e24abe5c performs an ordered sequence of indirect calls, arithmetic-derived memory accesses, narrow stores, pointer-mediated updates, and a final four-word publication.

No application-level purpose is established. The evidence does not identify an allocator, serializer, collection operation, or conventional object initializer. “Publication” here describes stores, not a synchronization or ownership guarantee.

The visible path ends at an invalid-instruction boundary, not an established normal return. Every later action is conditional on earlier calls returning, earlier accesses succeeding, and intervening arithmetic permitting continuation. Static evidence alone does not establish that the complete path is executable.

## Interface

### Incoming state and results

The recovered interface declares no explicit parameters or result, but consumes incoming machine state:

- Register 3 supplies an unsigned 32-bit address operand and values derived by logical right shifts.
- Registers 5 and 6 supply 32-bit address bases.
- The incoming link register participates in the first indirect-call target.
- Thirteen recovered stack-associated locations supply 32-bit words.

The stack-associated offsets are hexadecimal: **054, 168, 1CC, 208, 210, 248, 278, 2B4, 2F4, 374, 38C, 3C8, 3DC**. Below, **context[374]**, for example, denotes the recovered word associated with offset 374. This notation does not establish a public frame layout, its reference point, or when physical stack loads occur.

The words at offsets 210, 248, and 278 are represented as signed 32-bit quantities. Offset 168 supplies an uninterpreted 32-bit argument. The remaining context words are represented as unsigned 32-bit quantities.

No success value, error value, or normal-return convention is established. Secondary register state following the last call supplies a final byte store; this is not evidence of a structured function result.

### Widths and arithmetic

Direct accesses are 8, 16, or 32 bits. Byte reads are unsigned. A narrow store retains the low 8 or 16 bits of its value.

Unsigned right shifts are logical. Unsigned 32-bit arithmetic uses modulo-\(2^{32}\) interpretation in the representation. Signed arithmetic must be distinguished from that rule:

- F-U14’s signed result is reduced by hexadecimal 35.
- The signed register-6-relative loaded word is increased by hexadecimal 31.
- Those adjusted quantities are multiplied as signed values before the result is represented as unsigned.
- The adjusted F-U14 result is doubled, then much later quadrupled.
- The final context-derived value uses signed addition of hexadecimal 31.
- F-U12 receives a signed result and its four-bit left shift.

Out-of-range signed calculations and affected address expressions are **documented_only**; portable wrapping behavior is not established. Conversion of the product to unsigned does not independently resolve overflow in the preceding signed multiplication.

The represented address width is 32 bits. Byte order, alignment requirements, address mappings, and hardware fault behavior are unknown.

## Behavior

### Notation and ordering

All numerical displacements below are hexadecimal unless explicitly identified as decimal. Descriptive quantities are:

| Quantity | Meaning |
|---|---|
| **Register region** | Incoming register 3 logically shifted right by four |
| **Initial quarter-base** | context[2F4] logically shifted right by two |
| **Context anchor** | context[3DC] logically shifted right by two |
| **Anchor quarter** | Context anchor logically shifted right by two |
| **Adjusted factor** | Signed F-U14 result minus 35 |
| **Doubled factor** | Adjusted factor multiplied by two |
| **Product** | Unsigned representation of the signed product described below |
| **Selector byte** | Unsigned byte subsequently read at Context anchor + 1F |
| **Scale-64** | Selector byte multiplied by decimal 64 |
| **Scale-256** | Selector byte multiplied by decimal 256 |

Tables list represented accesses in order. They do not establish a hardware memory model. Within each pointer-mediated operation, the relative order of its two reads is unresolved.

### Initial call and memory inputs

The first visible call is F-U01, targeting an address derived from the incoming link register plus FFE. If it returns, the following accesses occur:

| Order | Access | Address | Value |
|---|---|---|---|
| 1 | 32-bit store | Register 3 + 34 | Register 3 shifted right by two |
| 2 | 32-bit store | Register region + 24 | Register 3 shifted right by six |
| 3 | 32-bit store | Register region + 24 | Same value as order 2 |
| 4 | 32-bit store | Initial quarter-base + 14 | Initial quarter-base |
| 5 | 32-bit store | Register region + 18 | Register 3 shifted right by six |
| 6 | 32-bit store | Initial quarter-base + 14 | context[374] shifted right by two |
| 7 | Unsigned byte read | Register region + 1B | Retained as the first byte |
| 8 | Signed 32-bit read | Register 6 + context[374] shifted right by two + 25 | Retained arithmetic input |

The byte read lies inside the word beginning at Register region + 18. Its value can therefore depend on that earlier store and byte order without requiring any additional aliasing.

Orders 2 and 3 are separate identical writes. Orders 4 and 6 are separate writes with another store between them. Final memory contents alone do not describe these access histories.

F-U02 receives context[374] shifted right by two, plus 25—the offset used by order 8. Calls F-U03 through F-U10 follow, then F-U11. F-U12 receives F-U11’s signed result and that result shifted left by four. F-U13 follows, then F-U14 supplies the result used to form the Adjusted factor.

The Product uses the Adjusted factor multiplied by the signed word from order 8 increased by 31. The Doubled factor is then formed. These calculations precede the following stores.

### Product-dependent stores

The next fifteen accesses are 16-bit stores. “Low half” means the low 16 bits after the stated arithmetic or shift.

| Order | Destination | Stored value |
|---|---|---|
| 1 | Product + 0A | First byte + 33 |
| 2 | Product + 0C | Low half of Product |
| 3 | Product + 0E | Low half of Product |
| 4 | Product shifted right by two + 0E | Low half of Product |
| 5 | Product + 12 | Low half of Product |
| 6 | Doubled factor + 12 | Low half of Product |
| 7 | context[278] + 14 | Low half of Product |
| 8 | Product + 16 | Low half of Product |
| 9 | Register 5 + 16 | Low half of Product |
| 10 | Product + 18 | Low half of Product |
| 11 | Product + 1A | Low half of context[2B4] shifted right by two |
| 12 | Doubled factor + 1C | Low half of Product |
| 13 | context[3DC] + 20 | Low half of context[208] shifted right by two |
| 14 | context[1CC] shifted right by two + 22 | Low half of context[1CC] shifted right by two |
| 15 | context[054] shifted right by two + 24 | Low half of context[3DC] |

The first addition is wider than a byte: decimal 255 plus hexadecimal 33 produces decimal 306, not decimal 50.

Next, a **pointer-mediated update** reads the 32-bit location at Doubled factor + 3C twice. One read supplies a destination base; the other supplies the stored 32-bit value. The destination is that base plus 18. The two reads are not guaranteed to agree if the source changes, and their relative evaluation order is unknown.

The subsequent 32-bit stores are:

| Destination | Value |
|---|---|
| Product + 3C | Context anchor |
| context[38C] + 18 | context[38C] shifted right by two |
| context[3C8] + 18 | context[3C8] shifted right by two |
| context[3DC] + Doubled factor | context[3DC] |
| Context anchor + 14 | Context anchor |

F-93c4ce5fe2dd then receives the Context anchor. After its return, its 32-bit result is stored at four times context[3DC] + 14.

Only after that result store does the routine read the Selector byte at Context anchor + 1F.

### Selector-driven accesses and calls

F-U15 receives the Selector byte and Context anchor. After its return:

| Width | Destination | Value |
|---|---|---|
| 32 | Context anchor + 34 | Anchor quarter |
| 32 | Context anchor + 18 | Anchor quarter |
| 32 | Anchor quarter + 14 | Anchor quarter |
| 16 | Four times context[3DC] | Low half of Anchor quarter |

F-U16 then receives two equal arguments, each four times the Selector byte. After its return:

| Width | Destination | Value |
|---|---|---|
| 32 | Context anchor + 18 | Anchor quarter |
| 32 | Selector byte shifted right by two + 24 | Anchor quarter |
| 32 | Anchor quarter + 14 | Anchor quarter |
| 32 | Selector byte shifted right by two + 18 | Scale-64 |
| 16 | Scale-64 + 04 | Low half of Scale-64 |
| 32 | Selector byte + 18 | Scale-256 |
| 16 | Context anchor + 04 | Low half of Anchor quarter |
| 32 | Scale-64 + 24 | Anchor quarter |
| 32 | Scale-64 + 24 | Anchor quarter |

The last two rows are consecutive distinct writes. Masks present in the selector-derived calculations do not change the Selector byte, because its range is already decimal 0–255. Scale-64 is therefore at most decimal 16,320; Scale-256 is at most decimal 65,280. These particular calculations do not overflow 32 bits.

F-U17 receives Scale-256 and Scale-64. Its returned 32-bit value is stored at four times Scale-256 + 14. Three more accesses follow:

| Width | Destination | Value |
|---|---|---|
| 16 | Anchor quarter + 02 | Low half of Anchor quarter |
| 32 | Scale-64 + 24 | Anchor quarter |
| 16 | Scale-64 + 04 | Low half of Scale-64 |

F-U18 receives Scale-256 and Scale-64.

**Immediately after F-U18 returns, a separate 16-bit store writes the low half of Anchor quarter to Scale-256 + 06.** This write precedes the second pointer-mediated update. It remains an independent observable effect under aliasing; it is not part of the later word store and must not be omitted.

The pointer-mediated update through Doubled factor + 3C is then repeated, with two fresh represented reads and the same unresolved internal read ordering.

The following 32-bit stores occur afterward:

| Destination | Value |
|---|---|
| Four times Scale-256 + 3C | Scale-64 |
| context[210] + 20 | Scale-64 |
| Scale-64 + 14 | Scale-64 |
| Scale-64 + 24 | context[374] shifted right by two |

The last row overwrites an earlier anchor-derived value at that destination, subject to aliasing and intervening dependency effects.

### Final publication and terminal boundary

F-7d50b94821c2 receives the Selector byte multiplied by decimal 16 and the 32-bit context[168] value. No result is explicitly consumed.

After that call returns, the final operations have this order:

1. Read an unsigned byte at context[374] shifted right by two, plus 18. This is the **final selector**, distinct from the earlier Selector byte.
2. Calculate the **final context value** as signed context[248] + 31.
3. **Quadruple the Adjusted factor using signed arithmetic.**
4. Store byte value decimal 100 at context[248] + 45.
5. Read a 32-bit destination pointer at context[248] + A5.
6. Store the final context value as a 32-bit word at final selector + 38.
7. Perform the four-word publication below.
8. Call F-U19.
9. Store one byte from post-call secondary register state at the quadrupled Adjusted factor + 1E.
10. Reach the invalid-instruction boundary.

The quadrupling is not deferred until the final byte store. It occurs after the final unsigned-byte read and final context-value calculation, but before the byte-100 store, pointer read, publication, and F-U19. Its unresolved signed out-of-range behavior therefore affects continuation before those later effects.

The final selector selects an absolute address between hexadecimal 38 and 137; its destination is not relative to the byte’s source address.

At the loaded destination pointer, four consecutive 32-bit stores occur in ascending order:

| Byte displacement | Value |
|---|---|
| 00 | Decimal 100 |
| 04 | Final context value |
| 08 | Final context value |
| 0C | Loaded pointer’s own value |

F-U19 receives decimal 100, the final context value, and Scale-64 + 25.

Neither the four-word layout nor storing the pointer itself establishes allocation, ownership, object identity, or a valid enclosing record. The terminal boundary establishes neither a normal return nor a particular processor exception or recovery path.

## Side effects

The routine modifies memory through incoming, loaded, shifted, and multiplied address operands. No bounds checks, null checks, ownership checks, synchronization, rollback, or compensating updates are visible. Zero does not suppress these accesses.

Aliasing is unrestricted by the evidence. Writes can overlap other destinations, pointer sources, later byte-read locations, or dependency-visible memory. Partial-width stores can affect subsequent wider accesses, and word stores can determine subsequent byte reads.

The recovered context inputs do not prove a physical stack-load schedule or protection from aliasing. Already computed quantities and later memory reads must not be conflated.

A fault, arithmetic-related failure, or non-returning call may leave completed effects in place. No explicit error-reporting channel is visible.

No allocation or release operation is explicitly identified in this body. Dependencies may nevertheless allocate, release, retain pointers, modify globals, or change memory. No ownership transfer or dependency preservation guarantee is established.

## Dependencies

Unresolved identifiers name call sites, not proven distinct functions.

| Opaque ID | Established role |
|---|---|
| F-U01 | Initial indirect call; target derived from incoming link register + FFE |
| F-U02 | Receives the register-6-relative read offset |
| F-U03–F-U10 | Ordered intervening calls; no explicit arguments shown |
| F-U11 | Supplies the signed value passed to F-U12 |
| F-U12 | Receives that value and its four-bit left shift |
| F-U13 | Intervening call; no explicit arguments shown |
| F-U14 | Supplies the result reduced by 35 |
| F-93c4ce5fe2dd | Receives Context anchor; returned word is stored |
| F-U15 | Receives Selector byte and Context anchor |
| F-U16 | Receives two copies of four times Selector byte |
| F-U17 | Receives Scale-256 and Scale-64; returned word is stored |
| F-U18 | Receives Scale-256 and Scale-64 |
| F-7d50b94821c2 | Receives Selector byte times decimal 16 and context[168] |
| F-U19 | Receives decimal 100, final context value, and Scale-64 + 25 |

Except for F-U01, unresolved targets derive from fixed code-valued expressions. Their semantic identities and executability are unknown. “No explicit arguments shown” does not mean no machine-state inputs. Calling conventions, preservation rules, memory effects, ownership, and return behavior remain unresolved.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Repeated accesses:** If execution reaches the initial stores, Register region + 24 receives two consecutive identical 32-bit writes. One write with the same final contents is not equivalent at the represented-access level.

2. **Widening before truncation:** If the first byte is decimal 255, the first Product-relative halfword store writes hexadecimal 0132.

3. **Shift before truncation:** If context[2B4] is hexadecimal 00040004, its shifted value is 00010001 and the corresponding halfword store writes 0001.

4. **Post-F-U18 halfword:** If Selector byte is decimal 2 and Context anchor is hexadecimal 00040004, the immediate post-F-U18 store writes hexadecimal 0001 at hexadecimal 0206. It occurs before either source read of the second pointer-mediated update. If that store overlaps the pointer source, the later reads must reflect the applicable memory semantics, not a cached earlier value.

5. **Bounded selector scaling:** A Selector byte of decimal 255 yields Scale-64 = 16,320, Scale-256 = 65,280, and a first argument of 4,080 to F-7d50b94821c2.

6. **Zero still accesses memory:** A zero Selector byte still leads to attempted accesses at low addresses, including hexadecimal 18 and 24. Success depends on the target mapping.

7. **Late quadrupling boundary:** If the Adjusted factor is signed decimal 600,000,000, doubling is within signed 32-bit range but quadrupling is not. If execution reaches the final phase, the unresolved operation occurs after the final selector read and final context-value calculation. The contract does not guarantee that the byte-100 store or publication occurs afterward.

8. **Fresh pointer reads:** A change to Doubled factor + 3C between the two pointer-mediated updates affects the second update’s fresh reads. Within either update, agreement between the two reads is not guaranteed.

9. **Publication and interruption:** With valid destinations and defined preceding arithmetic, publication writes decimal 100, the final context value twice, and the loaded pointer value in that order. If F-U19 does not return, the final secondary-register byte store is not reached; earlier publication has no visible rollback.

## Uncertainties

The following remain **documented_only**:

- Executability of the recovered entry and complete path.
- Calling convention, context-location interpretation, and physical input-load timing.
- Byte order, alignment, mappings, memory types, and fault behavior.
- Out-of-range signed arithmetic, signed left shifts, and affected address calculations.
- Relative ordering of the two reads within each pointer-mediated update.
- Dependency implementations, preservation, ownership, pointer retention, and failure behavior.
- Meaning and availability of the final secondary-register byte.
- Behavior beyond the invalid-instruction boundary.

An initialization-like intent is possible but is not an acceptance requirement.

## Evidence

Established evidence is limited to the supplied static representation: access widths, unsigned byte reads, arithmetic dependencies, repeated stores, call ordering, the immediate post-F-U18 halfword store, the placement of late signed quadrupling, two resolved opaque dependency identities, and invalid-instruction/control-flow warnings.

This evidence does not establish a valid public API, application-level data structure, complete executable path, or hardware access trace.

No runtime tests have been performed. No execution trace, target memory map, dependency implementation, or independent calling-convention evidence was provided.
