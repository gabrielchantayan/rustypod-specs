<!-- Trusted provenance: F-f138d80dcad7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f138d80dcad7

## Purpose

The supplied static representation describes an ordered sequence of indirect calls, four word stores, and six byte stores, followed by an unconditional loop containing additional calls and one byte store per iteration.

No application-level purpose is established. The evidence does not establish object initialization, buffer processing, device control, or resource management. The contract therefore describes observable accesses, arguments, and state transitions without assigning those meanings.

All described progress is conditional on preceding accesses succeeding and preceding dependencies returning normally. The representation reports invalid instruction data and removal of unreachable regions. Its apparent control flow is static evidence, not proof of the original executable’s behavior.

## Interface

### Explicit inputs

| Input | Visible width | Established use |
|---|---|---|
| First argument | Unsigned 32 bits | Initial stored value, scaled store address, dependency argument, and masked lookup-address input. |
| Second argument | Opaque 32 bits | No visible use on the represented path. |
| Third argument | Unsigned 32 bits | Logically shifted right by seven and passed to a dependency. |
| Fourth argument | Unsigned 32 bits | Stored as a word; also used to derive a byte-store address and dependency argument. |

An incoming link-register value also contributes to the first indirect call target. The four explicit arguments alone do not determine that target.

### Output and termination

No explicit return value or normal return is represented.

After the finite prefix, execution enters an unconditional loop. A dependency may fail to return, and an invalid access or indirect target may prevent further progress. No explicit validation, error status, exception handler, recovery path, or loop-exit condition is visible.

### Widths and arithmetic

Word accesses are 32 bits; byte accesses are eight bits. Lookup bytes are zero-extended before contributing to destination addresses.

Unsigned masks and shifts have 32-bit bit-pattern interpretations. Some dependency results and address intermediates are represented as signed 32-bit integers. Signed multiplication, signed left shifts, and conversion of an out-of-range unsigned value to a signed intermediate do not establish portable arithmetic behavior.

Where this contract gives a wrapping result, it is conditional on a 32-bit machine-bit-pattern interpretation. That interpretation must not be confused with a guarantee about overflowing signed expressions.

### Preconditions

To reach a particular effect:

- Each preceding indirect target must be callable under the actual ABI.
- Each preceding dependency must return normally.
- Each preceding memory access must be valid for its width and access type.
- The execution environment must support the represented address calculations.

No checks enforce these conditions. Endianness, alignment requirements, memory mapping, and fault delivery remain unspecified.

## Behavior

### Dependency notation and retained state

F-D01 through F-D60 are document-local opaque identifiers for **call sites**, ordered by first execution. They are not established external function identities. Different sites may resolve to the same function, and a repeatedly executed site need not retain the same target.

“No explicit arguments” describes the supplied representation only. It does not exclude implicit ABI inputs.

For readability:

- The **first lookup address** is one plus the first argument after clearing its highest bit and logically shifting right by seven.
- The **first packed result** is the unsigned 32-bit result consumed from F-D24.
- The **second packed result** is the unsigned 32-bit result consumed from F-D36.
- The **second lookup address** is derived from the second packed result as described below.

These are descriptive names, not assertions about data formats.

### 1. Initial word stores and calls

The first observable effect is a 32-bit store of the first argument at an address equal to four times that argument plus `0x14`.

The following calls occur afterward:

| Site | Explicit arguments | Consumed result |
|---|---|---|
| F-D01 | First argument | None |
| F-D02 | None | Opaque 32-bit value forwarded to F-D03 |
| F-D03 | F-D02 result | None |
| F-D04 | None | Opaque 32-bit value forwarded to F-D05 |
| F-D05 | F-D04 result | None |
| F-D06, F-D07, F-D08 | None, individually in order | None |
| F-D09 | None | Signed 32-bit index |

Using the F-D09 index, the routine stores the fourth argument as a 32-bit word at four times that index plus `0x14`. F-D10 then receives four times the same index.

These first two stores may alias. If they select the same address, the second store occurs after F-D01 through F-D09, whose effects are unknown.

### 2. Repeated word destination

F-D11, F-D12, and F-D13 execute without explicit arguments; their results are not consumed. F-D14 then supplies a signed 32-bit index.

The routine derives a scaled base equal to sixteen times that index and stores four times the index as a 32-bit word at the scaled base plus `0x14`.

F-D15 receives, in order:

1. The scaled base.
2. The same scaled base.
3. The third argument logically shifted right by seven.

The first lookup address is also derived during this phase. Its range is `0x00000001` through `0x01000000`, inclusive.

F-D16, F-D17, and F-D18 execute without explicit arguments and with unconsumed results. F-D19 then supplies a signed 32-bit result.

After F-D19 returns, the routine stores the scaled base itself as a 32-bit word at the same scaled-base-plus-`0x14` destination. This is a separate, observable store. Nothing establishes that the earlier contents survived the intervening calls.

F-D20 receives the F-D19 result shifted left by two, subject to the signed-shift qualification above.

### 3. First packed result and two selected byte writes

F-D21, F-D22, and F-D23 execute without explicit arguments; their results are not consumed. F-D24 supplies the first packed result.

The routine then:

1. Forms the unsigned 32-bit result of shifting the first packed result left by two.
2. Reads one byte at the first lookup address.
3. Adds `0x11` to the zero-extended byte.
4. Stores the low eight bits of the shifted result at the resulting address.

The loaded byte is a numeric address component, not a loaded pointer. Consequently, the selected destination lies between `0x11` and `0x110`, inclusive.

F-D25 receives three identical arguments containing the 32-bit shifted value. The representation retains that value through a signed intermediate; out-of-range signed conversion is not a portable-language guarantee.

Next:

| Site | Explicit arguments | Consumed result |
|---|---|---|
| F-D26 | None | Opaque 32-bit value forwarded to F-D27 |
| F-D27 | F-D26 result | None |
| F-D28, F-D29, F-D30 | None, individually in order | None |
| F-D31 | None | Signed 32-bit value retained for F-D32 |

After F-D31, the routine reads the first lookup byte **again**. At that newly selected byte-derived destination, it stores the low byte of the first packed result after masking with `0x3fffffff` and logically shifting right by twenty.

The stored byte is bits 20 through 27 of the first packed result. The masked, shifted intermediate can occupy ten bits; the byte store discards its upper two bits.

F-D32 then receives the F-D31 result shifted left by two.

### 4. Second packed result and two clearing writes

F-D33, F-D34, and F-D35 execute without explicit arguments and with unconsumed results. F-D36 supplies the second packed result.

The following effects occur in order:

1. Store the low byte of the second packed result shifted left by two at the fourth argument logically shifted right by seven, plus `0x1c`.
2. Read the byte at the first lookup address.
3. Write zero at that byte’s zero-extended value plus `0x11`.
4. Call F-D37 with the fourth argument logically shifted right by seven.
5. Call F-D38 without explicit arguments; its result is not consumed.
6. Call F-D39 with the second packed result masked by `0x3fffffff` and logically shifted right by six.

The fourth-argument-derived store address ranges from `0x1c` through `0x0200001b`. F-D39 receives a value between zero and `0x00ffffff`; this argument is not truncated to a byte.

F-D40, F-D41, and F-D42 then execute without explicit arguments, with unconsumed results. F-D43 supplies a signed 32-bit result.

After F-D43 returns, the routine again reads the byte at the first lookup address and writes zero to that freshly selected byte-derived destination. F-D44 then receives the F-D43 result shifted left by two.

Each lookup is a distinct read. Dependencies or earlier aliased writes may change its value. In particular, the fourth-argument-derived byte store occurs before the next first-lookup read and can affect that read if their addresses coincide.

### 5. Final pre-loop clearing write

F-D45, F-D46, and F-D47 execute without explicit arguments and with unconsumed results. F-D48 supplies an unsigned 32-bit result.

The initial loop field is that result masked by `0x3fffffff`, then logically shifted right by 25. Its range is zero through 31.

The routine derives the second lookup address from the second packed result. The represented calculation retains its left-by-two shifted value in a signed intermediate, multiplies that intermediate by `0x400`, and adds one.

Under 32-bit wrapping interpretation, the resulting address is the second packed result shifted left by twelve, plus one. It ranges over addresses whose low twelve bits equal one, from `0x00000001` through `0xfffff001`. The signed conversion and multiplication prevent treating that wrapping interpretation as an unconditional portable arithmetic guarantee.

The routine reads one byte at the second lookup address and stores zero at the byte’s zero-extended value plus `0x11`.

F-D49 receives the initial loop field shifted left by one: an even value from zero through 62. F-D50 then executes without explicit arguments. Its signed 32-bit result becomes the initial carried loop value.

### 6. Unconditional loop

Every iteration has this observable order:

1. F-D51 receives the carried loop value and the current loop field logically shifted right by eight.
2. F-D52, F-D53, and F-D54 execute without explicit arguments; their results are not consumed.
3. F-D55 executes without explicit arguments and supplies a signed 32-bit result.
4. The routine reads one byte at the second lookup address.
5. It writes zero at that byte’s zero-extended value plus `0x11`.
6. F-D56 receives the F-D55 result shifted left by two.
7. F-D57, F-D58, and F-D59 execute without explicit arguments; their results are not consumed.
8. F-D60 executes without explicit arguments and supplies a signed 32-bit result.
9. The next carried loop value is the F-D60 result shifted left by two.
10. The next loop field is the F-D60 result shifted left by 26, retained as unsigned 32-bit data.

The first F-D51 call receives zero as its second argument because the initial loop field is at most 31.

Under the 32-bit bit-pattern interpretation, later F-D51 second arguments equal the low six bits of the preceding F-D60 result shifted left by eighteen. This recurrence differs from the initial five-bit extraction.

The target expression at F-D51 uses different retained address state on the first iteration and subsequent iterations. The representation therefore does not support assuming one stable target for all executions of that site. This distinction does not change the represented argument order.

The second lookup address remains derived from the second packed result, but its byte is re-read on each iteration. No exit condition is represented.

## Side effects

### Direct memory effects

For an execution that completes the finite prefix:

- **Four 32-bit stores** occur.
- **Six byte stores** occur.
- **Five of those six byte stores use byte-derived destinations.**
- Five lookup-byte reads occur: four at the first lookup address and one at the second.

The six pre-loop byte stores, in order, are:

| Order | Stored byte | Destination |
|---|---|---|
| 1 | Low byte of the first packed result shifted left by two | First lookup byte plus `0x11` |
| 2 | Bits 20–27 of the first packed result | Fresh first lookup byte plus `0x11` |
| 3 | Low byte of the second packed result shifted left by two | Fourth argument shifted right by seven, plus `0x1c` |
| 4 | Zero | Fresh first lookup byte plus `0x11` |
| 5 | Zero | Fresh first lookup byte plus `0x11` |
| 6 | Zero | Second lookup byte plus `0x11` |

Each completed loop iteration adds one lookup-byte read and one byte store. An interrupted iteration may stop before either access.

These are access counts, not counts of distinct addresses or changed values. Repeated stores remain observable even when they write the same value to the same destination. Word and byte destinations may alias lookup locations or each other wherever the address arithmetic permits.

Word byte order is unknown. No alignment checks or synchronization operations are represented.

### Dependency effects, ownership, and errors

Every dependency may have additional effects. No dependency is established to preserve global memory, avoid I/O, allocate or release memory, retain an argument, or terminate.

There is no established direct allocation, deallocation, ownership transfer, or resource-release operation. Forwarding a returned 32-bit value does not prove that it is a pointer or that ownership changes.

No explicit error channel exists. Faults and invalid indirect transfers are environmental execution failures rather than represented return statuses. No recovery obligations are visible.

## Dependencies

F-D01 through F-D60 identify call sites only. Their implementations, external identities, and full signatures are unavailable.

Established dependency information is limited to:

- Execution order relative to other calls and direct accesses.
- Explicit argument counts and ordering.
- Which returned values are consumed.
- The represented widths and subsequent operations on those values.
- The incoming link-register contribution to F-D01’s target.
- The first-versus-later target distinction at F-D51.

A visibly unconsumed result does not make its producing call behaviorally removable.

All additional dependency semantics are **documented_only**, including memory preservation, error conventions, implicit inputs, ownership, and target equivalence.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests.

1. **Initial access ordering.**  
   With first argument 3 and a valid destination, the first effect is a 32-bit store of 3 at address `0x20`. F-D01 follows that store.

2. **Lookup masking.**  
   First arguments `0x00000080` and `0x80000080` both select first lookup address 2. This does not make their initial stored values or all other behavior equal.

3. **Byte-derived address width.**  
   A lookup value of `0xff` selects destination `0x110`. No word-sized pointer is loaded from the lookup address.

4. **First packed-result extraction.**  
   If F-D24 returns `0x12345678`, the first selected byte store writes `0xe0`; the later selected byte store writes `0x23`. Fresh lookup reads may direct them to different addresses.

5. **Lookup aliasing affects the next read.**  
   If the fourth argument is zero, the direct byte destination is `0x1c`. If the first argument is `0x00000d80`, the first lookup address is also `0x1c`. With F-D36 returning 1, the direct store writes 4 there before the next lookup reads it. Absent another writer, that lookup selects zero-store destination `0x15`.

6. **Exact pre-loop store accounting.**  
   If execution reaches F-D49, it has performed four word stores and six byte stores. Five byte destinations came from loaded byte values; one came from the fourth argument. The loop has not yet contributed a byte store.

7. **Repeated reads may select different destinations.**  
   If successive relevant lookup reads return 4 and 9, their selected destinations are `0x15` and `0x1a`. Reusing the earlier byte would not match the represented accesses.

8. **Initial loop field.**  
   If F-D48 returns all one bits, the initial loop field is 31, F-D49 receives 62, and the first F-D51 second argument is zero.

9. **Later loop transition.**  
   Under 32-bit bit-pattern semantics, if F-D60 returns 1, the next carried value is 4 and the next loop field is `0x04000000`. The next F-D51 second argument is `0x00040000`.

10. **Failure limits subsequent obligations.**  
    If a lookup read faults, its dependent byte store and subsequent effects are not reached on that execution. No represented handler resumes the sequence.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the reconstructed control flow, given the reported invalid instruction data and removed regions.
- Actual dependency identities, signatures, side effects, and return behavior.
- ABI rules and interpretation of implicit machine state.
- Exact signed-overflow, signed-shift, and out-of-range conversion behavior.
- Memory mapping, endianness, alignment requirements, and fault handling.
- Whether low numeric destinations are ordinary memory, mapped registers, or inaccessible addresses.
- Concurrent modification of lookup bytes or other memory.
- Whether distinct call targets reach equivalent behavior.
- Any application-level meaning of indices, packed results, or clearing operations.
- Any ownership or lifetime convention.
- Effects of the second argument outside the represented behavior.

An initialization or ongoing state-maintenance role would be inferred intent, not an established purpose. No such interpretation is required by this contract.

## Evidence

Established static evidence consists of the supplied four-argument representation, its ordered indirect calls, its explicit memory accesses and arithmetic, and its unconditional terminal loop.

Direct enumeration establishes four pre-loop word stores and six pre-loop byte stores. Five of the byte stores use loaded-byte-derived destinations. The loop contributes one further byte store per iteration that reaches that access.

The masks, shifts, truncations, repeated reads, and retained loop state support the conditional examples above. Signed-arithmetic limitations are stated where relevant rather than resolved by assumption.

The representation’s warnings limit confidence in executable reconstruction; they do not establish alternative behavior. No runtime tests have been performed.
