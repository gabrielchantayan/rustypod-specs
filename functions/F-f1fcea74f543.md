<!-- Trusted provenance: F-f1fcea74f543; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f1fcea74f543

## Purpose

The available static representation describes a routine that interleaves indirect calls with address-dependent word and byte accesses. It accepts one explicit 32-bit value, forwards that value to its first indirect call, and later performs arithmetic on selected call results to form addresses and arguments.

No application-level purpose is established. In particular, the representation does not justify identifying the accessed storage as an array, object, device register set, allocation, or serialization buffer.

The visible path ends at a bad-instruction boundary rather than a normal return. The representation also carries a control-flow reliability warning. Consequently, this contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **Documented-only unknowns:** properties that cannot be determined from that material.

The contract below describes the represented path conditionally: each later operation is reached only if earlier calls and memory accesses permit execution to continue.

## Interface

### Explicit input

The routine has one explicit input:

| Input | Width | Established use |
|---|---:|---|
| Input value | 32 bits | Passed unchanged as the sole explicit argument of the first indirect call. |

The input is not directly used by the later arithmetic shown in the representation. Its indirect effects through that first call remain unknown.

### Implicit execution inputs

The represented path also depends on:

- An incoming link/control-register value, used to form the first indirect-call target by adding `0xFFE`.
- An incoming register value whose producer is not visible, later written as a 32-bit word.
- The results and side effects of indirect calls.
- Memory contents at dynamically calculated addresses.
- The execution environment’s addressing, alignment, instruction-state, and exception rules.

The incoming register value is separate from the explicit input. No evidence establishes that they contain the same value.

### Output and termination

The representation declares no ordinary return value. It shows no normal return instruction or completed return path.

Its visible terminal action is a bad-instruction termination marker. Whether execution in the original environment traps, enters some other instruction stream, or behaves differently is documented_only. A successful normal return must not be promised.

### Width and layout conventions

For the arithmetic descriptions below:

- **Word** means the represented 32-bit access.
- **Byte** means an unsigned 8-bit read or an 8-bit write.
- Byte reads are zero-extended when used in address calculations.
- `0x14` is a byte offset of 20.
- `0x11` is a byte offset of 17.
- Right shifts of the explicitly unsigned intermediate values are logical right shifts.

The representation does not establish byte order for word accesses, valid address ranges, or alignment behavior.

Expressions involving overflowing signed multiplication or signed left shifts cannot safely be assigned high-level language semantics from this evidence alone. Where examples use low-32-bit arithmetic, that interpretation is stated as a condition rather than presented as an independently established machine property.

## Behavior

### Dependency notation and retained values

The indirect call sites receive local opaque identifiers **F-000000000001** through **F-000000000044**, in represented execution order. These identifiers name call sites, not verified distinct functions. Multiple sites may resolve to the same function.

The following semantic names identify values used across stages:

| Value | Origin |
|---|---|
| Incoming word | Implicit incoming register value |
| First index | Result of F-000000000005 |
| Second index | Result of F-000000000010 |
| Captured word | Word read after the second index is obtained |
| Third index | Result of F-000000000015 |
| Fourth index | Result of F-000000000020 |
| Fifth index | Result of F-000000000025 |
| Payload | Unsigned 32-bit result of F-000000000030 |
| Forwarded token | 32-bit result of F-000000000032 |
| Sixth index | Result of F-000000000037 |
| Second selector | Second byte read from the selector address |
| Seventh index | Result of F-000000000042 |

These names describe roles within this contract; they do not identify application types or ownership.

### 1. Initial dispatch and incoming-word store

F-000000000001 receives the explicit input value. Its target is derived from the incoming link/control-register value plus `0xFFE`.

F-000000000002, F-000000000003, and F-000000000004 then execute without explicit arguments in the representation.

F-000000000005 executes without explicit arguments and produces the first index.

The routine then writes the incoming word to the word address:

**4 × first index + 20**

F-000000000006 is called afterward with **4 × first index** as its sole explicit argument.

No bounds check, null check, or validation precedes this store in the represented path.

### 2. Second-index store followed by a separate word read

F-000000000007 through F-000000000009 execute without explicit arguments. F-000000000010 then supplies the second index.

Two memory operations follow, in this order:

1. Write **4 × second index** as a word at **16 × second index + 20**.
2. Read a word from **4 × second index + 20**, retaining it as the captured word.

The write address and read address use different scaling factors. The read must not be described as reading back the preceding store except when the actual addresses coincide.

F-000000000011 then receives **4 × second index**.

### 3. Shifted captured-word store

F-000000000012 through F-000000000014 execute without explicit arguments. F-000000000015 supplies the third index.

The routine writes:

**captured word logically shifted right by 12**

as a word at:

**4 × third index + 20**

The captured word is treated as unsigned for this shift. The stored result therefore lies between zero and `0x000FFFFF`.

F-000000000016 then receives **4 × third index**.

### 4. Fourth-index store and three-argument dispatch

F-000000000017 through F-000000000019 execute without explicit arguments. F-000000000020 supplies the fourth index.

The routine writes **4 × fourth index** as a word at:

**16 × fourth index + 20**

F-000000000021 then receives three explicit arguments, in order:

1. **((4 × third index) AND `0x3F`) shifted left by 14**
2. **4 × fourth index**
3. **third index shifted left by 30**

The first argument masks the scaled third index, not the captured word.

Under low-32-bit arithmetic, the first argument depends only on the lowest four bits of the third index. Its possible values are multiples of `0x10000`, ranging from zero through `0xF0000`. Under that same arithmetic interpretation, the third argument depends only on the lowest two bits of the third index.

### 5. Fifth-index store

F-000000000022 through F-000000000024 execute without explicit arguments. F-000000000025 supplies the fifth index.

The routine writes **4 × fourth index** as a word at:

**4 × fifth index + 20**

This reuses the scaled fourth index. It does not store the scaled fifth index.

F-000000000026 then receives **4 × fifth index**.

### 6. First selector read and payload-byte write

F-000000000027 through F-000000000029 execute without explicit arguments. F-000000000030 supplies the unsigned 32-bit payload.

The routine reads one unsigned byte from the selector address:

**third index × `0x10000000` + 1**

It then writes the low eight bits of **payload shifted left by 2** to:

**first selector byte + 17**

The selector byte is used as an absolute address component. The resulting write address ranges from 17 through 272; it is not shown as relative to a buffer base.

The written value is equivalently four times the payload’s lowest six bits. Its bottom two bits are always zero.

F-000000000031 follows with three explicit arguments:

1. **payload shifted left by 2**
2. **payload shifted left by 2**
3. **payload shifted left by 4**

The first two arguments are equal in the representation. This call site uses a target offset of `0xFEC`; the other represented call sites use `0xFFE`.

### 7. Token forwarding and second selector read

F-000000000032 produces a 32-bit forwarded token. F-000000000033 receives that token unchanged.

F-000000000034 through F-000000000036 execute without explicit arguments. F-000000000037 supplies the sixth index.

The routine then reads the selector address again:

**third index × `0x10000000` + 1**

This is a fresh byte read, not reuse of the first selector. Its zero-extended result is retained as the second selector.

The routine writes a byte at:

**second selector + 17**

The byte value is the low eight bits of:

**(payload AND `0x3FFFFFFF`) logically shifted right by 20**

Although the intermediate shifted result can occupy ten bits, the byte write retains only eight. The stored byte is therefore payload bits 20 through 27.

The selector may have changed between the two reads. In particular, intervening dependencies are not known to preserve memory. The first byte write can also itself target the selector address if the address calculations coincide.

### 8. Final dispatches and terminal boundary

F-000000000038 receives **sixth index shifted left by 2**.

F-000000000039 through F-000000000041 execute without explicit arguments. F-000000000042 supplies the seventh index.

F-000000000043 receives:

1. **seventh index shifted left by 2**
2. **second selector**, zero-extended from its byte read

There is no third selector read before this call.

F-000000000044 then executes without explicit arguments. The represented path subsequently reaches the bad-instruction termination marker.

## Side effects

### Direct memory access order

Excluding dependency-internal activity, the represented direct accesses occur in this order:

| Order | Access | Address | Value |
|---:|---|---|---|
| 1 | Word write | 4 × first index + 20 | Incoming word |
| 2 | Word write | 16 × second index + 20 | 4 × second index |
| 3 | Word read | 4 × second index + 20 | Captured word |
| 4 | Word write | 4 × third index + 20 | Captured word logically shifted right by 12 |
| 5 | Word write | 16 × fourth index + 20 | 4 × fourth index |
| 6 | Word write | 4 × fifth index + 20 | 4 × fourth index |
| 7 | Byte read | Third index × `0x10000000` + 1 | First selector |
| 8 | Byte write | First selector + 17 | Payload bits 0–5 shifted into bits 2–7 |
| 9 | Byte read | Third index × `0x10000000` + 1 | Second selector |
| 10 | Byte write | Second selector + 17 | Payload bits 20–27 |

The call placements described above are part of this ordering. The table does not imply that these accesses are adjacent or that dependencies have no intervening memory effects.

### Aliasing and state preservation

No non-aliasing guarantee is established. A later read can observe an earlier direct write if their actual byte ranges overlap. Dependencies can also modify any accessible state; their effects are unknown.

The captured word, payload, and second selector are reused as represented values. This does not establish that unknown dependencies preserve globals, memory, or any particular physical register.

### Ownership and resource handling

No direct allocation, deallocation, acquisition, or release operation is identifiable in the visible body. This does not establish that dependencies perform none.

The input and forwarded token are opaque 32-bit values. Neither may be classified as owned memory, borrowed memory, or a resource handle from the available evidence.

### Errors and preconditions

The represented path contains no explicit validation, error-result branch, rollback, retry, or recovery.

Reaching the terminal boundary requires all earlier indirect transfers to resolve and continue, and all preceding memory accesses to be permitted by the environment. Unmapped low addresses, invalid targets, alignment restrictions, or dependency failures may prevent later operations from occurring.

Earlier writes are not shown as rolled back if a later operation fails.

## Dependencies

All 44 dependency sites are indirect. Only F-f1fcea74f543 has a supplied resolved identity; the dependency identifiers below are local opaque site labels.

| Opaque dependency sites | Explicit argument/result role |
|---|---|
| F-000000000001 | Receives the routine input |
| F-000000000002–F-000000000004 | No explicit arguments; no result subsequently used |
| F-000000000005 | Produces first index |
| F-000000000006 | Receives scaled first index |
| F-000000000007–F-000000000009 | No explicit arguments; no result subsequently used |
| F-000000000010 | Produces second index |
| F-000000000011 | Receives scaled second index |
| F-000000000012–F-000000000014 | No explicit arguments; no result subsequently used |
| F-000000000015 | Produces third index |
| F-000000000016 | Receives scaled third index |
| F-000000000017–F-000000000019 | No explicit arguments; no result subsequently used |
| F-000000000020 | Produces fourth index |
| F-000000000021 | Receives masked/scaled third index, scaled fourth index, and high-shifted third index |
| F-000000000022–F-000000000024 | No explicit arguments; no result subsequently used |
| F-000000000025 | Produces fifth index |
| F-000000000026 | Receives scaled fifth index |
| F-000000000027–F-000000000029 | No explicit arguments; no result subsequently used |
| F-000000000030 | Produces payload |
| F-000000000031 | Receives two equal shifted payload arguments and a third differently shifted argument |
| F-000000000032 | Produces forwarded token |
| F-000000000033 | Receives forwarded token |
| F-000000000034–F-000000000036 | No explicit arguments; no result subsequently used |
| F-000000000037 | Produces sixth index |
| F-000000000038 | Receives shifted sixth index |
| F-000000000039–F-000000000041 | No explicit arguments; no result subsequently used |
| F-000000000042 | Produces seventh index |
| F-000000000043 | Receives shifted seventh index and second selector |
| F-000000000044 | Final represented dependency call |

“No explicit arguments” describes only the supplied representation. Hidden register inputs, calling conventions, memory inputs, and additional results are documented_only.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Different scaling for the second-index write and read

If the second index is 3, arithmetic is exact, and both accesses are valid:

- The word write targets address 68 and stores 12.
- The following word read targets address 32.
- The captured word comes from address 32, not from the newly written word at address 68.

An implementation that reads address 68 instead would not match the represented behavior.

### Same-address store and read

If the second index is zero and address 20 is valid, both the word write and following word read target address 20.

With ordinary stable memory and no intervening external modification, the write stores zero and the read captures zero. This example demonstrates that the distinct formulas do not guarantee distinct addresses.

### Unsigned captured-word shift

If the captured word is `0xFEDCBA98`, the subsequent shifted word value is `0x000FEDCB`.

The high bits are filled with zero, not copied from the original top bit.

### Masked three-argument dispatch

If the third index is 5 and the fourth index is 7, using low-32-bit shift semantics, F-000000000021 receives:

1. `0x00050000`
2. 28
3. `0x40000000`

The first argument derives from the scaled third index; it does not derive from the captured word.

### Payload byte extraction

If the payload is `0x0AB0003F`:

- The first byte write stores `0xFC`.
- The second byte write stores `0xAB`.

These values follow from distinct portions of the same retained payload.

### Selector changes between reads

If the first selector read returns 7 and the second returns 9:

- The first byte write targets address 24.
- The second byte write targets address 26.
- F-000000000043 receives 9 as its second argument.

Reusing 7 for the second write or final argument would not match the represented behavior.

### Intervening dependency changes the selector

If an intervening dependency modifies the selector byte, the second selector read observes memory at its later read point. The routine provides no basis for assuming selector stability.

### Inaccessible early destination

If the first word-store destination is inaccessible, no later dependency call or memory operation is guaranteed. No represented recovery path permits execution to skip that store and continue.

## Uncertainties

### Established static evidence

The supplied representation explicitly supports:

- One explicit 32-bit input.
- Forty-four ordered indirect call sites.
- Five direct word writes, one word read, two byte reads, and two byte writes.
- The listed address formulas, masks, shifts, and result forwarding.
- A terminal bad-instruction boundary.

### Inferred intent

The repeated scaling and fixed offsets may suggest indexed storage or encoded state. The payload extraction may suggest bit-field handling. Neither interpretation establishes a data structure, protocol, or application purpose.

### Documented-only unknowns

The following remain unresolved:

- Whether the represented path is executable as shown.
- Actual dependency identities and calling conventions.
- Whether dependencies return normally or modify relevant state.
- Address width and precise overflow behavior.
- Meaning of signed arithmetic where values overflow or are shifted outside a safely defined range.
- Endianness and alignment requirements.
- Memory mapping and access permissions, especially for low absolute addresses.
- Whether any accessed location has device or other nonordinary memory semantics.
- Ownership and lifetime of all dependency-associated resources.
- The original behavior beyond the bad-instruction boundary.

## Evidence

This contract is based solely on the supplied static representation and its control-flow warnings.

The explicit accesses, call ordering, argument expressions, and retained results provide the static evidence. No implementation execution, runtime tests, dependency inspection, or independent instruction-level validation has been performed.

The acceptance examples are arithmetic consequences under their stated conditions, not observed runtime outcomes.
