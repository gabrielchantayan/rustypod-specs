<!-- Trusted provenance: F-8de3be456a6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8de3be456a6d

## Purpose

F-8de3be456a6d performs a fixed visible sequence of indirect calls, byte-mediated writes, 32-bit stores, and bit-field transformations. Dependency results supply most addresses and payloads. Additional entry state, absent from the four explicit inputs, supplies the first call target and initial byte access.

No application-level purpose is established. The available evidence does not identify an object format, protocol, device, allocator, or ownership model.

The supplied static representation ends at an invalid-instruction boundary, not a normal return. This contract therefore describes an execution prefix: each operation is conditional on preceding calls and memory accesses allowing execution to continue. It does not guarantee completion or a recoverable error result.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Passed to F-U001. |
| Second | 32 bits | No explicit use in the visible body. |
| Third | 32 bits | No explicit use in the visible body. |
| Fourth | Signed 32 bits | Shifted left by 10; the transformed value supplies two arguments to F-U013 and one word-store payload. |

The second and third inputs have no visible dataflow use. This does not establish that their entry-machine locations are irrelevant to hidden arguments or unresolved calling conventions.

### Additional effective inputs

Execution also depends on:

- Incoming execution-link state used to form the first indirect target.
- An incoming address-like value used for a byte read at offset 1.
- An incoming byte value used as the first byte-store payload.
- Execution-location-derived values used to form later call targets.
- Memory accessed through addresses computed from dependency results.

These are requirements of the supplied representation, not established public parameters.

### Outputs and completion

No explicit return value or normal return is visible. Observable outputs consist of calls and direct memory effects.

F-U014 supplies a represented 64-bit result. Its upper and lower 32-bit words are used independently. Other consumed results are represented as 32-bit values. Nothing establishes the physical register arrangement or byte order used to deliver the wide result.

The invalid-instruction boundary does not establish a specific exception, exit status, recovery mechanism, or externally visible error code. Those outcomes are **documented_only**.

### Widths, arithmetic, and layout

A byte is 8 bits; a word is 32 bits. Byte reads used in address calculations are zero-extended. Right shifts applied to unsigned words are logical.

Relevant layouts are limited to:

- A byte source at an address-like value plus 1.
- Byte destinations at the zero-extended source byte plus 17.
- Word destinations derived by scaling a selector and adding 20.
- Word sources at a derived base plus 20.

No surrounding allocation, record size, element type, or buffer boundary is established.

Unsigned 32-bit operations retain their low 32 bits. Some address calculations and left shifts instead have signed represented operands. Their overflow behavior cannot be given portable signed-language semantics from this evidence. Numerical examples explicitly assume 32-bit bit-vector arithmetic, including truncation of left shifts and products.

Word-store byte order, access alignment, pointer conversion, and effective address wrapping remain **documented_only**.

## Behavior

### Sequencing convention

F-U001 through F-U098 identify indirect call sites in visible execution order. They are opaque documentation IDs, not resolved dependency identities. Different IDs may reach the same implementation.

“No explicit arguments” describes only the supplied representation. It does not establish that a machine-level call has no hidden arguments.

The phases below preserve direct memory-access and call ordering. Retained values refer to visible dataflow, not an assumption that dependencies preserve arbitrary registers, globals, or memory.

### 1. Initial byte-mediated write

F-U001 receives the first explicit input. F-U002 through F-U004 then execute without explicit arguments. F-U005 supplies a 32-bit result.

After F-U005, the function reads one byte from the incoming address-like value plus 1. It writes the incoming byte payload to the numerical address equal to that read byte plus 17. The destination consequently lies between 17 and 272 inclusive; no additional destination base is represented.

F-U006 receives the F-U005 result shifted left by 2.

F-U007 through F-U009 execute without explicit arguments, followed by the result-producing F-U010. F-U011 receives, in order:

- The initially captured byte, zero-extended to 32 bits.
- The F-U010 result shifted left by 2.

The initial byte is not reread for F-U011.

### 2. Fourth-input transformation and wide-result use

F-U012 executes without explicit arguments. The fourth input is shifted left by 10, and F-U013 receives that transformed value in both explicit argument positions.

F-U014 then supplies the 64-bit result. Its upper word determines the destination of a word store: four times the upper word, plus 20. The stored payload is the transformed fourth input. This store precedes F-U015, which receives the upper word.

After F-U016 through F-U018, F-U019 supplies a selector. A second word store writes the upper word at four times this selector plus 20. F-U020 receives four times the selector.

These are two separate stores. Their destinations need not be distinct. The lower word from F-U014 is retained for later operations.

### 3. Lower-word transformations

After F-U021 through F-U024, a word store uses:

- Destination: the lower word multiplied by \(2^{28}\), plus 20.
- Payload: the lower word multiplied by \(2^{26}\).

F-U025 receives the same multiplication-by-\(2^{26}\) value.

The lower word is also masked to its low 29 bits. After F-U026 through F-U028, F-U029 supplies a selector. A word store at four times that selector plus 20 receives the masked value logically shifted right by 9. F-U030 receives four times the selector.

The extracted payload consists of original lower-word bits 9 through 28. Its range is 0 through \(2^{20}-1\).

### 4. First derived base and two word-read expressions

After F-U031 through F-U033, F-U034 supplies a selector. Four times that selector becomes a derived base. The function stores this base at sixteen times the selector plus 20.

F-U035 receives three explicit arguments:

1. A word read from the derived base plus 20, logically shifted right by 12.
2. The derived base.
3. A word read from the derived base plus 20, shifted left by 2.

There are two word-read expressions at the same address. Both follow the preceding word store and supply arguments to F-U035, but their relative evaluation order is unresolved. The evidence does not justify replacing them with one cached read or requiring them to return equal values.

After F-U036 through F-U038, F-U039 supplies another selector. The retained derived base is stored at four times this new selector plus 20. F-U040 receives four times the new selector.

### 5. Second derived base

After F-U041 through F-U043, F-U044 supplies a selector. Four times this selector becomes a new derived base and is stored at sixteen times the selector plus 20.

F-U045 receives the derived base twice, followed by the lower word from F-U014 shifted left by 26.

After F-U046 through F-U048, F-U049 supplies a selector. The retained base is stored at four times this selector plus 20. F-U050 receives four times the selector.

### 6. Third derived base

After F-U051 through F-U053, F-U054 supplies a selector. Four times this selector is stored at sixteen times the selector plus 20 and passed to F-U055.

After F-U056 through F-U058, F-U059 supplies another selector. The same retained four-times-selector value is stored at four times the new selector plus 20. F-U060 receives four times the new selector.

### 7. Fourth derived base and captured word

After F-U061 through F-U063, F-U064 supplies a selector. Four times this selector becomes the next derived base and is stored at sixteen times the selector plus 20.

The function then reads one unsigned word from the derived base plus 20. This capture occurs before F-U065, which receives the derived base.

After F-U066 through F-U068, F-U069 supplies a selector. The captured word, logically shifted right by 12, is stored at four times this selector plus 20. F-U070 receives four times the selector.

The source word is not reread for this store. Later changes to the source memory do not change the already captured payload.

### 8. Selector-derived bit arguments

After F-U071 through F-U073, F-U074 supplies a selector. Four times this selector is stored at sixteen times the selector plus 20.

F-U075 receives:

1. The low six bits of four times the F-U069 selector, shifted left by 14.
2. Four times the F-U074 selector.
3. The F-U069 selector shifted left by 30.

Under 32-bit bit-vector arithmetic, the first argument depends only on the low four bits of the F-U069 selector; the third depends only on its low two bits.

After F-U076 through F-U078, F-U079 supplies another selector. Four times the F-U074 selector is stored at four times the F-U079 selector plus 20. F-U080 receives four times the F-U079 selector.

### 9. First late byte write

After F-U081 through F-U083, F-U084 supplies a 32-bit result retained for both late byte payloads.

The late byte source address is the F-U069 selector multiplied by \(2^{28}\), plus 1. A byte read there selects the destination: its zero-extended value plus 17.

At that destination, the function writes the low byte of the F-U084 result shifted left by 2. F-U085 then receives:

- The F-U084 result shifted left by 2.
- The same shifted value.
- The F-U084 result shifted left by 4.

F-U085 has a different target-forming displacement from the regular displacement used by most other call sites. The difference is part of the visible behavior; it does not establish a resolved target identity.

### 10. Second late byte write and terminal calls

F-U086 supplies a 32-bit value passed directly to F-U087. F-U088 through F-U090 execute without explicit arguments, followed by result-producing F-U091.

After F-U091, the function rereads the same late byte source address used in phase 9. The newly read byte selects a destination at that byte plus 17. The payload is the low byte of the retained F-U084 result after masking to 30 bits and logically shifting right by 20.

This is a new read across intervening calls. It need not select the same destination as the first late byte write. The first write or a dependency could also have affected the source, if addresses alias.

F-U092 receives the F-U091 result shifted left by 2. After F-U093 through F-U095, F-U096 supplies another result. F-U097 receives that result shifted left by 2, followed by the byte captured by the second late read, zero-extended.

F-U098 executes without explicit arguments. The representation then reaches the invalid-instruction boundary.

## Side effects

The complete visible prefix contains:

| Direct effect | Count |
|---|---:|
| Byte stores | 3 |
| 32-bit word stores | 14 |
| **All direct stores** | **17** |
| Byte reads | 3 |
| Word-read expressions | 3 |
| Indirect calls | 98 |

The fourteen word stores are the two stores in each of phases 2 through 8. Phases 1, 9, and 10 each contribute one byte store. The three word-read expressions comprise the two argument reads for F-U035 and the single capture before F-U065.

These counts exclude dependency-internal effects and do not promise that execution reaches every operation.

No explicit allocation, release, ownership transfer, synchronization, validation, or rollback is established. Values passed to dependencies are not thereby proven to be pointers, handles, or owned objects.

Direct access destinations may alias each other, source locations, or dependency-accessible state. Later reads observe memory at their execution points; captured values remain the values already read. No dependency is assumed to leave memory or globals unchanged.

Execution requires suitable readable, writable, and executable locations. The visible body provides no checks establishing those conditions. If a call or access prevents continuation, earlier effects have already occurred; no visible mechanism reverses them. Exact fault and error behavior remains **documented_only**.

## Dependencies

All 98 indirect call sites remain unresolved. Their actual interfaces, hidden arguments, memory effects, preservation rules, ownership behavior, termination, and error signaling are unknown.

Most targets are formed from an execution-location-derived value plus 4094. F-U001 instead uses incoming execution-link state plus 4094. F-U085 uses displacement 4076.

Consumed results have the following roles:

| Dependency IDs | Visible role |
|---|---|
| F-U005, F-U010 | Sources for early shifted arguments. |
| F-U014 | Wide result supplying independently used upper and lower words. |
| F-U019, F-U029 | Selectors for early word destinations. |
| F-U034, F-U044, F-U054, F-U064, F-U074 | Selectors establishing derived bases. |
| F-U039, F-U049, F-U059, F-U069, F-U079 | Selectors for subsequent word destinations; F-U069 also controls later bit arguments and byte-source addressing. |
| F-U084 | Retained source of both late byte payloads and F-U085 arguments. |
| F-U086 | Value forwarded to F-U087. |
| F-U091, F-U096 | Sources for final shifted arguments. |

Other call results have no explicit use. Their calls remain observable and cannot be treated as removable: their effects and control behavior are unknown.

## Acceptance examples

These are conditional arithmetic and sequencing examples, not executed tests. Unless stated otherwise, they assume returning dependencies, valid accesses, and 32-bit bit-vector arithmetic.

1. **Initial byte destination.** An initial byte read of 255 selects numerical destination 272. A read of 0 selects destination 17. Neither establishes a surrounding buffer.

2. **Fourth-input scaling.** A fourth-input bit pattern of 1 yields 1024 after the shift. F-U013 receives 1024 twice. If F-U014’s upper word is 3, the next word store writes 1024 at address 32.

3. **Truncating shift.** A fourth-input bit pattern of `0x00400000` becomes zero after a 32-bit left shift by 10. This is conditional on bit-vector behavior, not a guarantee about signed overflow on an unspecified target.

4. **Independent wide-result halves.** With upper word 3 and lower word 1, the phase-2 destination is 32. The first phase-3 store instead targets `0x10000014` and writes `0x04000000`.

5. **Field extraction.** A lower word of `0xffffffff` produces `0x000fffff` for the store selected by F-U029. Original bits 29–31 and 0–8 do not contribute.

6. **Two reads need not agree.** If memory changes between the two F-U035 argument reads, the first and third arguments may derive from different words. Acceptance cannot require a particular read order or equal underlying values.

7. **Captured word remains captured.** If phase 7 reads `0x12345000`, its later shifted store writes `0x00012345`, even if a dependency subsequently changes the source memory.

8. **Late byte truncation.** If F-U084 returns `0x3ff00001`, the first late payload is `0x04`. The second is `0xff`: the mask-and-shift yields `0x3ff`, of which only the low eight bits are stored.

9. **Reread changes destination.** If the first late byte read yields 2 and the second yields 7, the writes target 19 and 24 respectively. F-U097 receives 7 as its second explicit argument.

10. **No successful-return implication.** Return from F-U098 only permits reaching the represented invalid-instruction boundary. It does not establish successful completion of F-8de3be456a6d.

## Uncertainties

### Established static evidence

The supplied representation establishes the visible call order, explicit argument relationships, widths, masks, direct access expressions, retained-value relationships, and terminal boundary. Its direct stores total seventeen: fourteen word stores and three byte stores.

### Inferred intent

Repeated scaling, field extraction, and offset-20 accesses suggest structured manipulation of encoded values or selectors. This is **inferred intent**, insufficient to identify a data structure or application purpose.

### Documented-only unknowns

The following remain **documented_only**:

- Whether the represented control flow and recovered interface faithfully describe executable behavior.
- Actual dependency identities, contracts, and hidden arguments.
- Preservation of machine state across unresolved calls beyond the depicted dataflow.
- Signed-overflow behavior, pointer conversion, and address wrapping.
- Word byte order, alignment constraints, and memory permissions.
- Relative order of the two F-U035 word reads.
- Concurrency, volatile-memory, and device-register semantics.
- Object lifetime, ownership, and aliasing guarantees.
- Access-failure behavior, exception handling, and continuation beyond the terminal boundary.

These uncertainties prevent an unconditional portable implementation contract.

## Evidence

This contract is derived solely from the supplied static representation. That representation reports invalid instruction data and truncated control flow, limiting claims about actual execution.

The effect inventory follows the explicit accesses: two word stores in each of seven middle phases, three byte stores, three byte reads, and three word-read expressions. Dependency IDs identify the 98 visible indirect call sites, not verified implementations.

No implementation access, runtime execution, or tests were performed. Acceptance examples are conditional consequences of the described dataflow and arithmetic assumptions, not observed outcomes.
