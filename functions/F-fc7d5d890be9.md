<!-- Trusted provenance: F-fc7d5d890be9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc7d5d890be9

## Purpose

F-fc7d5d890be9 describes an ordered execution prefix containing indirect dependency calls, computed byte accesses, and 32-bit memory writes. The supplied static representation does not establish an application-level purpose.

Visible effects include publishing the third input twice, clearing computed fields, storing scaled dependency results, and writing byte fragments extracted from inputs or dependency results.

The represented path ends at an invalid-instruction boundary. Normal completion, a returned status, cleanup, and a stable final memory image are not established. This contract specifies the visible prefix, including the ordering of effects before any interruption.

## Interface

### Explicit inputs and output

| Input | Represented width | Visible use |
|---|---:|---|
| First input | Unsigned 32 bits | Address calculations, shifts, masks, and dependency arguments |
| Second input | Opaque 32 bits | No direct use shown |
| Third input | Opaque 32 bits | Stored unchanged by two 32-bit writes |

No ordinary return value is represented. The accuracy of the apparent function boundary and calling interface remains **documented_only**.

The second input’s absence from explicit expressions does not establish machine-level irrelevance: incomplete dependency signatures leave ambient argument-state observations unresolved.

### Implicit inputs

Execution also depends on:

- An incoming register value used to form the first write address.
- An incoming link-related value used to form the first indirect-call target.
- An additional register result observed after F-D0806.
- Bytes and a 32-bit payload read from computed addresses.
- Dependency results, memory effects, register effects, and control-flow behavior.

No visible validation establishes that these inputs are suitable.

### Widths and arithmetic

Byte accesses transfer 8 bits. Word accesses transfer 32 bits. F-D0805 has a represented 64-bit result, subsequently separated into low and high 32-bit halves.

The following descriptive terms are used:

| Term | Meaning |
|---|---|
| Scaled input | Four times the first input |
| Group | First input logically shifted right by three |
| Masked group | Group masked with `0x003fffff` |
| Group-table offset | Group multiplied by `0x400` |
| Bank offset | Group multiplied by `0x04000000` |

Unsigned 32-bit arithmetic in the representation reduces modulo `2^32`. In particular, the group-table and bank multiplications can wrap before their results are used as signed address intermediates.

Other calculations involve represented signed 32-bit values, including dependency results, left shifts, and address arithmetic. Their behavior on signed overflow, and the interpretation of out-of-range unsigned-to-signed conversions, require target-specific rules. The formulas below describe the represented relationships without imposing a universal wrapping rule on every operation.

A **selector-directed write** comprises two accesses in order:

1. Read an unsigned byte from the specified selector address.
2. Write one byte at the selector value plus `0x11`.

The destination is therefore between `0x11` and `0x110`, inclusive. The selector read does not load a pointer-sized address.

### Preconditions and ownership

Reaching the terminal boundary requires every preceding access and dependency call to complete. Computed addresses must satisfy the target’s mapping, permissions, and alignment requirements; indirect targets must be callable with the actual machine state.

These are execution preconditions, not checked conditions.

No allocation, release, ownership transfer, or lifetime extension is established. Unknown dependencies may retain arguments or affect storage; neither preservation nor release may be assumed.

## Behavior

The phases below are ordered. A later phase is reached only when preceding operations complete. Each named dependency ID identifies a call occurrence, not a verified distinct implementation.

Unless stated otherwise, a dependency has no explicit arguments represented. This does not establish a zero-argument machine interface. Results described as 32-bit are represented values, not independently verified dependency signatures.

### 1. Initial publication and payload chain

Write the third input as a 32-bit word at the incoming register value multiplied by `0x80000`, plus `0x24`.

Invoke F-D0101 with the scaled input and first input, in that order. Its target is formed from the incoming link-related value plus `0xffe`. Invoke F-D0102 through F-D0105 in ascending order.

Write the third input as a 32-bit word at the scaled input plus `0x24`.

Read one 32-bit payload at the scaled input plus `0x74`, and pass it to F-D0201. Invoke F-D0202 through F-D0205 in ascending order. Retain F-D0205’s 32-bit result as the **early result**.

The two publication writes remain separate events even if their addresses coincide.

### 2. First selector-directed updates

Read the selector at the scaled input plus one. Write the low eight bits of the first input logically shifted right by 22 to that selector’s destination.

Invoke F-D0301 with the early result shifted left by two. Invoke F-D0302 and pass its 32-bit result to F-D0303. Invoke F-D0304 through F-D0307 in ascending order, retaining F-D0307’s result.

Read the selector at the scaled input plus one again, and write zero to its destination. This is a fresh read, not reuse of the first selector.

Invoke F-D0401 with F-D0307’s retained result shifted left by two. Invoke F-D0402 through F-D0405 in ascending order.

This phase contains two selector-byte reads and two byte writes.

### 3. Group-directed byte clearing

Write zero directly to the byte at the group plus `0x0f`.

Read the selector at the group-table offset plus one, and write zero to its destination.

Invoke F-D0501 with four arguments:

1. The early result shifted left by 23.
2. The same shifted value.
3. The same shifted value.
4. The early result shifted left by 25.

Invoke F-D0502 and pass its 32-bit result to F-D0503. Invoke F-D0504 through F-D0507 in ascending order, retaining F-D0507’s result.

Read the selector at the group-table offset plus one again. Retain that byte and write zero to its destination.

Invoke F-D0601 with F-D0507’s result shifted left by two. Invoke F-D0602 through F-D0605 in ascending order.

This phase contains two selector-byte reads and three byte writes. The direct clear at group plus `0x0f` has no associated selector read.

### 4. First word-field clearing

Using the selector retained in phase 3, write a 32-bit zero at its value plus `0x44`. Then write a 32-bit zero at absolute address `0x24`.

The retained selector is not reread after the intervening dependencies.

Invoke F-D0701 with these arguments, in order:

1. Zero.
2. `0x98`.
3. The group-table offset.
4. `0x98`.

Invoke F-D0702 and pass its 32-bit result to F-D0703. Invoke F-D0704 and pass its 32-bit result to F-D0705. Invoke F-D0706 through F-D0709 in ascending order, retaining F-D0709’s result.

Write a 32-bit zero at four times that result, plus `0x44`. Then write a 32-bit zero at absolute address `0x24`.

### 5. Split result and additional register result

Invoke F-D0801 with four times F-D0709’s result. Invoke F-D0802 through F-D0805 in ascending order.

F-D0805 produces a represented 64-bit result. Invoke F-D0806 with its low 32-bit half shifted left by two.

Write a 32-bit zero at twice the high 32-bit half, plus `0x44`. Then write a 32-bit zero at absolute address `0x24`.

Invoke F-D0901 with:

1. Zero.
2. The additional register result exposed after F-D0806.
3. That same register result shifted left by two.

The additional register result is not established as either half of F-D0805’s result.

Invoke F-D0902. Invoke F-D0903 with the masked group shifted left by seven. Invoke F-D0904 through F-D0907 in ascending order, retaining F-D0907’s result.

Write a 32-bit zero at four times that result, plus `0x44`. Then write a 32-bit zero at absolute address `0x24`.

Phases 4 and 5 contain four separate clears of absolute address `0x24`. Their ordering relative to dependency calls is observable; they cannot be collapsed into one final clear.

### 6. Clearing fields at offset `0x14`

Invoke F-D1001 with four times F-D0907’s result. Invoke F-D1002 through F-D1005 in ascending order.

Invoke F-D1006 with F-D1005’s result shifted left by two, retaining F-D1006’s result.

Write a 32-bit zero at four times F-D1006’s result, plus `0x14`.

Invoke F-D1101 with four arguments:

1. F-D1006’s result.
2. The same result.
3. The same result.
4. The masked group shifted left by seven.

Invoke F-D1102 through F-D1105 in ascending order. Write a 32-bit zero at four times F-D1105’s result, plus `0x14`.

### 7. Linking scaled results

Invoke F-D1201 with four times F-D1105’s result. Invoke F-D1202 through F-D1205 in ascending order.

Retain four times F-D1205’s result as the **saved scaled result**. Write that value as a 32-bit word at sixteen times F-D1205’s result, plus `0x14`.

Invoke F-D1301 with the saved scaled result. Invoke F-D1302 through F-D1305 in ascending order.

Write the saved scaled result as a 32-bit word at four times F-D1305’s result, plus `0x14`. The stored value remains derived from F-D1205.

Invoke F-D1401 with four times F-D1305’s result. Invoke F-D1402 through F-D1405 in ascending order.

Write four times F-D1405’s result as a 32-bit word at sixteen times that same result, plus `0x14`.

These arithmetic relationships do not establish linked-list, object-table, or ownership semantics.

### 8. Masked-group publication

Invoke F-D1501 with four times F-D1405’s result and sixteen times that result, in that order.

Invoke F-D1502. Pass its 32-bit result to F-D1503 as the first argument; the second argument is the masked group shifted left by four.

Invoke F-D1504 through F-D1507 in ascending order.

Write the masked group multiplied by `0x80` as a 32-bit word at four times F-D1507’s result, plus `0x14`.

The masked group is at most `0x003fffff`. Its shift by four and its multiplication by `0x80` fit within 32 bits; the latter is at most `0x1fffff80`.

### 9. Bank-selected byte fragments

Invoke F-D1601 with four times F-D1507’s result. Invoke F-D1602 through F-D1605 in ascending order. Retain F-D1605’s unsigned 32-bit result as the **first fragment source**.

Read the selector at the bank offset plus one. Write the low eight bits of the first fragment source shifted left by two to its destination.

Invoke F-D1701 with the first fragment source shifted left by two. Invoke F-D1702 and pass its 32-bit result to F-D1703. Invoke F-D1704 through F-D1707 in ascending order.

Read the selector at the bank offset plus one again. Write the low eight bits of the first fragment source, masked with `0x3fffffff` and logically shifted right by 20, to its destination.

Invoke F-D1801 with F-D1707’s result shifted left by two. Invoke F-D1802 through F-D1805 in ascending order. Retain F-D1805’s unsigned 32-bit result as the **second fragment source**.

Read the selector at the bank offset plus one a third time. Retain this selector and write the low eight bits of four times the second fragment source to its destination.

This phase contains three selector-byte reads and three byte writes. The separately sampled selectors need not identify the same destination.

### 10. Final byte update and boundary

Invoke F-D1901 with the selector retained in phase 9, zero-extended to 32 bits. Its argument is the selector value, not the fragment written and not the destination address.

Invoke F-D1902 and pass its 32-bit result to F-D1903. Invoke F-D1904 through F-D1907 in ascending order.

Read a selector byte at eight times the second fragment source, plus one. Write the low eight bits of the second fragment source, masked with `0x3fffffff` and logically shifted right by 20, to its destination.

Invoke F-D2001 with F-D1907’s result shifted left by two. Invoke F-D2002 through F-D2005 in ascending order.

This phase contains one selector-byte read and one byte write.

The representation then reaches an invalid-instruction boundary. No normal return or subsequent cleanup is established.

## Side effects

### Direct memory accesses

For the complete visible prefix:

| Direct operation | Count |
|---|---:|
| 32-bit writes of the third input | 2 |
| Byte writes | 9 |
| 32-bit zero writes at computed locations with offset `0x44` | 4 |
| 32-bit zero writes to absolute address `0x24` | 4 |
| 32-bit zero writes at computed locations with offset `0x14` | 2 |
| Value-dependent 32-bit writes at locations with offset `0x14` | 4 |
| Selector-byte reads | 8 |
| 32-bit payload reads | 1 |

The byte-access totals follow directly from the phase breakdown:

| Phase | Selector-byte reads | Byte writes |
|---|---:|---:|
| 2 | 2 | 2 |
| 3 | 2 | 3 |
| 9 | 3 | 3 |
| 10 | 1 | 1 |
| **Total** | **8** | **9** |

Thus, the visible prefix has sixteen direct word writes and nine direct byte writes. Counts describe access events, not distinct addresses, and exclude dependency-internal accesses.

Computed addresses may coincide or partially overlap. A byte write may modify storage later read as a selector or payload. Word writes may overlap byte destinations or other word writes. Endianness and access granularity are not established sufficiently to specify every overlapping byte outcome.

### Ordering, interruption, and state

Each selector read precedes its associated byte write. Repeated selector reads remain separate accesses, even when their address expressions match.

Dependencies are interleaved with direct accesses and may alter memory or other state. No preservation of globals, argument-referenced storage, or ownership is assumed.

No validation, rollback, error translation, error return, or synchronization operation is visible. An access fault, invalid call, nonreturning dependency, or other interruption may prevent later effects. No visible operation undoes earlier completed effects.

The stated order does not establish hardware barriers, atomicity, or cross-thread visibility guarantees.

## Dependencies

F-D identifiers are opaque occurrence labels. Distinct labels do not prove distinct targets.

| Call family | Occurrences | Significant represented result handling |
|---|---|---|
| Initial | F-D0101–F-D0105 | No explicit result consumed |
| Payload | F-D0201–F-D0205 | Final result retained |
| First fragment | F-D0301–F-D0307 | Second result forwarded; final result consumed |
| First clearing | F-D0401–F-D0405 | No explicit result consumed |
| Group fragment | F-D0501–F-D0507 | Second result forwarded; final result consumed |
| Group clearing | F-D0601–F-D0605 | No explicit result consumed |
| First word | F-D0701–F-D0709 | Two intermediate results forwarded; final result consumed |
| Split result | F-D0801–F-D0806 | 64-bit result split; additional register subsequently observed |
| Masked group | F-D0901–F-D0907 | Final result consumed |
| First `0x14` clear | F-D1001–F-D1006 | Fifth result forwarded; sixth result consumed |
| Second `0x14` clear | F-D1101–F-D1105 | Final result consumed |
| First linking | F-D1201–F-D1205 | Scaled final result retained |
| Second linking | F-D1301–F-D1305 | Final result selects destination |
| Third linking | F-D1401–F-D1405 | Final result supplies value and destination |
| Group publication | F-D1501–F-D1507 | Second result forwarded; final result consumed |
| First bank | F-D1601–F-D1605 | Final unsigned result retained |
| Bank forwarding | F-D1701–F-D1707 | Second result forwarded; final result consumed |
| Second bank | F-D1801–F-D1805 | Final unsigned result retained |
| Final selector | F-D1901–F-D1907 | Second result forwarded; final result consumed |
| Terminal | F-D2001–F-D2005 | No explicit result consumed |

Unconsumed explicit results do not imply absence of register, memory, device, ownership, or control-flow effects.

## Acceptance examples

These are conditional obligations derived from static evidence, not executed tests. Each assumes that preceding calls and accesses complete and that the relevant target arithmetic and addresses are valid.

1. **Unsigned selector.** A selector of `0xff` directs a byte write to `0x110`. It is not sign-extended and is not treated as a pointer loaded from memory.

2. **First-input truncation.** For first input `0x40000000`, phase 2 writes fragment `0x00`: shifting right by 22 produces `0x100`, whose low byte is zero. This does not establish validity of the input-derived addresses.

3. **Fresh versus retained selector.** If phase 2 first reads `0x20`, its first write targets `0x31`. If the selector becomes `0x21` before the second read, the zero write targets `0x32`. Conversely, a phase-3 retained selector of `0x20` selects the phase-4 word-clear address `0x64`, despite later changes to its source byte.

4. **Saved linking value.** If F-D1205 returns 3 and F-D1305 returns 9, phase 7 writes 12 at `0x44`, then 12 at `0x38`. The second stored value is not 36.

5. **Split result.** If F-D0805’s high half is 5 and low half is 3, F-D0806 receives 12 and the following computed word clear targets `0x4e`. Alignment validity remains target-dependent. The additional register result is still unspecified.

6. **Fragment extraction.** For first fragment source `0x12345678`, the two corresponding phase-9 byte values are `0xe0` and `0x23`. Independently sampled selectors may direct them to different locations.

7. **Selector argument.** If phase 9’s third selector is `0x7f`, F-D1901 receives `0x0000007f`, regardless of the fragment byte written to `0x90`.

8. **Access accounting.** If the complete prefix executes, it performs nine direct byte writes and eight selector-byte reads. The extra byte write is the direct phase-3 clear at group plus `0x0f`; it does not introduce another read.

9. **Interrupted prefix.** If F-D0703 does not return, F-D0704 and subsequent direct accesses are not reached. Earlier effects are not visibly rolled back.

## Uncertainties

The following remain **documented_only**:

- The true function boundary and machine-level calling interface.
- Exact dependency identities, signatures, conventions, and return widths.
- Validity and origin of incoming register-derived state.
- Meaning of the additional register result after F-D0806.
- Signed-overflow, signed-shift, conversion, address-width, and alignment rules.
- Endianness, atomicity, and overlapping-access behavior.
- Whether low addresses identify ordinary memory, devices, or invalid mappings.
- Dependency effects on globals, storage, ownership, and control flow.
- Whether invalid instruction data reflects executable behavior or an inaccurate static boundary.
- Behavior beyond the terminal boundary.

Repeated clears and scaled-field writes suggest initialization or state reconstruction. That is inferred intent only; no application data structure, ownership model, or successful postcondition is established.

## Evidence

Established static evidence is limited to the supplied representation: input and access widths, explicit arithmetic, dependency occurrence order, argument forwarding, retained values, direct memory operations, and the terminal invalid-instruction boundary.

The nine byte writes are individually accounted for as two in phase 2, three in phase 3, three in phase 9, and one in phase 10. Eight of those writes use explicit selector reads; the remaining write directly clears group plus `0x0f`. The separate payload read is 32 bits and occurs once in phase 1.

The supplied material warns about invalid instruction data and overlapping global storage. Those warnings constrain confidence; they do not justify repairing control flow or inventing additional behavior.

No implementation access, independent instruction validation, runtime execution, or tests have been performed. Acceptance examples are conditional static specifications, not observed outcomes.
