<!-- Trusted provenance: F-37cbbe76db78; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-37cbbe76db78

## Purpose

F-37cbbe76db78 describes an ordered sequence of indirect calls, byte and 32-bit memory accesses, and transformations of call results. No application-level purpose is established. In particular, the supplied static representation does not justify interpreting these operations as allocation, initialization, rendering, serialization, or cleanup.

The visible endpoint is an invalid-instruction termination marker, not a normal return. This contract describes the represented path conditionally: a preceding call, memory fault, invalid target, or arithmetic ambiguity may prevent later operations from being reached.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | Supplied to F-A01. |
| Second argument | 32 bits | No explicit use appears. |
| Third argument | 32 bits | Stored once as a full word and twice as a byte. |
| Fourth argument | Signed 32 bits | Shifted left by one and reduced to a byte for one store. |

The lack of a represented use of the second argument does not establish that an actual indirect callee cannot observe its incoming machine location.

### Additional entry state

Four additional incoming values participate in the represented path:

- **Entry base:** a signed 32-bit value used in address calculations and byte stores.
- **Entry selector:** an unsigned 32-bit value used to select an early byte-store destination.
- **Entry address seed:** an unsigned 32-bit value used to derive a memory base.
- **Entry link:** a value used to form the target of F-A01.

These labels describe data dependencies, not object types or validated hidden parameters. Their initialization and machine-level preservation are unresolved.

### Outputs, widths, and arithmetic

No normal return value is established. Observable outputs consist of memory writes, dependency effects, and any externally visible consequence of the abnormal endpoint.

Byte reads yield unsigned values from 0 through 255. Byte writes retain eight bits. Word accesses are 32 bits. F-A07, F-F09, and F-G01 have represented 64-bit results; their low and high halves are each 32 bits.

Explicitly unsigned right shifts are logical. Unsigned 32-bit multiplication and left shifts retain the low 32 bits in the represented arithmetic. Subsequent conversion of an unsigned result into a signed 32-bit quantity can introduce interpretation uncertainty. Signed doubling, signed left shifts, signed multiplication, and signed address additions are not assigned a universal wrapping rule by this contract.

Endianness, alignment requirements, atomicity, and address-space validity are unknown. Offsets describe accesses, not established structure layouts.

## Behavior

### Ordering and dependency notation

Each opaque dependency ID identifies a call occurrence. Different IDs do not necessarily identify different executable functions. A call described without arguments has no explicit arguments in the supplied representation; this is not proof of an empty machine-level input set.

The order below includes memory accesses and explicitly retained calculations. Calls may change relevant memory or global state. Repeated reads therefore remain separate reads, and repeated writes remain separate effects.

### A. Entry calls and first stores

F-A01 receives the first argument at an entry-link-derived target. F-A02 follows and supplies a 32-bit result. The entry address seed is then logically shifted right by two to obtain **seed base**, before F-A03 receives the F-A02 result.

F-A04, F-A05, and F-A06 occur in that order. F-A07 then supplies a represented 64-bit result.

Two stores precede F-A08:

- At seed base + 1, write the low byte of entry base.
- At entry base + 0x34, write the high 32-bit half of the F-A07 result.

F-A08 receives the low half of that result shifted left by two, with the represented signed interpretation of that half. Its signed-shift behavior is unresolved for values outside the defined arithmetic domain.

F-A07 and the later F-E07 have the **same numerical indirect-call target**. Their occurrence IDs remain distinct because their arguments, result treatment, and positions differ.

### B. Entry-derived byte writes

F-B01, F-B02, and F-B03 precede F-B04, whose 32-bit result is retained. Entry base is then doubled as a signed calculation, producing **doubled base**.

The next two effects are byte stores: first the low byte of doubled base at the entry selector logically shifted right by 22, plus 0x11; then the low byte of the third argument at doubled base + 0x0d.

F-B05 receives two identical arguments, each the F-B04 result shifted left by two. F-B06 supplies an unsigned 32-bit result. F-B07 receives that result followed by its logical right shift by three.

After F-B08, F-B09, and F-B10, F-B11 supplies another 32-bit result. The function writes the low byte of doubled base at absolute address 0x11, then again writes the third argument’s low byte at doubled base + 0x0d. F-B12 receives the F-B11 result shifted left by two.

The two third-argument byte writes are not interchangeable with one write: dependency calls separate them.

### C. Seed-based and scaled storage

F-C01, F-C02, and F-C03 precede F-C04, which supplies an unsigned 32-bit **scale result**.

The third argument is written as a full 32-bit word at seed base + 0x14. The low byte of seed base shifted left by one is then written at seed base + 0x0d.

Before F-C05, three derived values are formed, in this order:

- **Scaled base:** scale result multiplied by 32 in unsigned 32-bit arithmetic, then held in a signed 32-bit representation.
- **Scaled word:** scale result shifted left by seven in unsigned 32-bit arithmetic, then held in a signed 32-bit representation.
- **Reduced selector:** scale result masked with 0x07ffffff and logically shifted right by 17, yielding 0–1023.

F-C05 and F-C06 precede F-C07, which supplies a signed 32-bit **shift source**. The scaled word is written at scaled base + 0x14. The low byte of shift source shifted left by two is written at scaled base + 0x0d. F-C08 receives shift source shifted left by two.

F-C09, F-C10, F-C11, and F-C12 follow. The next operations have this specific order:

1. Write the low byte of the fourth argument shifted left by one at reduced selector + 7.
2. Shift the signed shift source left by 21 and retain the resulting **wide-shift value**.
3. Read an unsigned byte at scaled base + 1.
4. At that byte value + 0x11, write the low byte of reduced selector.
5. Call F-C13 with the retained wide-shift value as all three arguments.

The wide-shift calculation occurs **before** the selector-byte read and selected write. Because its signed-shift semantics are unresolved for some inputs, the contract does not move it across those accesses or substitute a later calculation.

The selected destination ranges from absolute address 0x11 through 0x110.

### D. Fresh selection, clearing, and retained payload byte

F-D01 supplies an unsigned 32-bit **payload**. F-D02 receives it. F-D03, F-D04, and F-D05 precede F-D06, whose 32-bit result is retained.

An unsigned byte is freshly read at scaled base + 1 and retained as **saved selector**. Zero is written at saved selector + 0x11. F-D07 then receives the F-D06 result shifted left by two.

After F-D08, F-D09, F-D10, and F-D11, the function derives:

- **Selector base:** twice saved selector, ranging from 0 through 510.
- **Saved payload byte:** the low byte of payload logically shifted right by three.

The saved payload byte is written at scaled base + 0x0d. Saved selector is then written at absolute address 0x0d. F-D12 receives selector base, 0x98, and 0x260, in that order.

This selector read is independent of the earlier section-C read. Once saved, its value is reused without rereading scaled base + 1 for selector-base calculations.

### E. Auxiliary register input and selector-based storage

F-E01 precedes F-E02. F-E02 receives an auxiliary unsigned 32-bit register value logically shifted right by three. The representation exposes that value but does not establish a complete producer or return-value contract for it.

F-E03, F-E04, and F-E05 precede F-E06, which supplies a 32-bit result. F-E07 receives that result shifted left by two and itself supplies a represented 32-bit result.

F-E07 uses the same numerical target as F-A07, although F-A07’s result is represented as 64 bits. This establishes neither a common return convention nor the correctness of either recovered result width.

The saved payload byte is written at selector base + 0x0d, followed by 0xc0 at absolute address 0x0d. F-E08 receives the F-E07 result shifted left by two.

F-E09, F-E10, and F-E11 precede F-E12, which supplies a signed 32-bit **link result**. Its left shift by two is written as a word at selector base + 0x24. A 32-bit unsigned word is then read at selector base + 0x74 and retained as **loaded base**. Only after that read is **link word** formed by shifting link result left by five.

F-E13 receives, in order:

- Loaded base.
- Loaded base shifted left by one.
- Link word.
- Loaded base masked with 0x7fffffff, then logically shifted right by two.

No validation of loaded base appears before its later address uses.

### F. Loaded-base writes and a paired result

F-F01, F-F02, F-F03, and F-F04 precede a word write of link word at loaded base + 0x24.

A 32-bit word is read at loaded base + 0x74 and passed to F-F05. F-F06, F-F07, and F-F08 follow, then F-F09 supplies a represented 64-bit result.

Its low half, interpreted as signed 32 bits and shifted left by two, becomes **paired word**. The high half is treated as unsigned 32 bits.

At the high half multiplied by four, plus 0x14, the function writes a full 32-bit word containing payload logically shifted right by three. This address calculation uses unsigned 32-bit arithmetic. F-F10 then receives paired word three times, followed by the high half logically shifted right by three.

The stored payload word is not restricted to the saved payload byte. Its maximum value is 0x1fffffff.

### G. Another paired result and indexed storage

F-G01 supplies another represented 64-bit result. F-G02 receives its low half, high half, low half again, and high half logically shifted right by three.

F-G03, F-G04, and F-G05 precede F-G06, whose signed 32-bit result is **word index**. Paired word from section F is written at word index multiplied by four, plus 0x14. F-G07 receives word index multiplied by four. These multiplications use a signed operand; overflow behavior is unresolved.

F-G08, F-G09, and F-G10 precede F-G11, which supplies an unsigned 32-bit **final scale result**. Its unsigned left shift by two is retained in a signed 32-bit representation as **final scaled value**.

An unsigned byte is read at loaded base + 1. At that byte value + 0x11, the low byte of final scaled value is written.

### H. Final selected writes and endpoint

F-H01 and F-H02 precede F-H03, whose 32-bit result is retained.

A fresh unsigned byte is read at loaded base + 1. At that byte value + 0x11, the function writes the low byte of final scale result masked with 0x3fffffff and logically shifted right by 20. The intermediate masked-and-shifted value ranges from 0 through 1023; the store retains only its low eight bits.

F-H04 receives the F-H03 result shifted left by two. F-H05, F-H06, and F-H07 precede F-H08, whose 32-bit result is retained.

The function reads an unsigned byte at final scaled value multiplied by 0x400, plus 1, then writes zero at that byte value + 0x11. The multiplication uses the signed retained final scaled value and has unresolved overflow behavior.

F-H09 receives, in order, final scaled value shifted left by three, the same shifted value again, final scaled value, and the F-H08 result shifted left by two.

F-H10, F-H11, and F-H12 precede F-H13. The indexed byte read and selected zero write are then repeated using a fresh read, not the previously obtained byte. F-H14 receives the F-H13 result shifted left by two.

F-H15, F-H16, F-H17, and F-H18 occur in order. The represented path then reaches the invalid-instruction endpoint.

## Side effects

The function performs byte writes, word writes, and indirect calls without visible rollback.

Storage includes absolute byte addresses 0x0d and 0x11; byte-selected destinations from 0x11 through 0x110; and base-relative offsets +1, +7, +0x0d, +0x14, +0x24, +0x34, and +0x74. These offsets do not establish a shared object layout.

Addresses may alias. A write can affect a later read, including a selector read. The two writes to absolute address 0x0d—saved selector, then 0xc0—are ordered effects, not a guarantee of a final value: later dependencies or aliasing writes may change that location.

No allocation, release, ownership transfer, synchronization, or persistence operation is established locally. Dependencies may perform such operations; neither ownership nor preservation of storage is assumed.

Progress requires every reached access and call target to be valid in the execution environment. No local recovery path, error result, or rollback is visible for invalid memory, invalid targets, or callee failure. Earlier effects may remain observable.

## Dependencies

All listed dependency IDs denote indirect-call occurrences with unavailable implementations.

Established properties are their represented arguments, result uses, ordering, and the numerical target equality between F-A07 and F-E07. Equal numerical targets do not by themselves establish executable validity, instruction-state interpretation, function identity, or compatible ABI treatment.

F-A07 is represented as producing 64 bits, while F-E07 is represented as producing 32 bits from the same numerical target. This discrepancy remains part of the static evidence, not a resolved interface.

Other target identities are not established here. A result with no explicit subsequent use can still accompany arbitrary callee effects. No dependency is assumed to return, preserve globals or registers, retain storage validity, or manage memory on the caller’s behalf.

## Acceptance examples

These are conditional static examples, not executed tests. Reaching each described operation assumes valid preceding accesses, returning dependencies, and applicable arithmetic semantics.

1. **Argument width.** For third argument 0x123456ab, both writes at doubled base + 0x0d carry byte 0xab. The write at seed base + 0x14 carries full word 0x123456ab.

2. **Ordered wide shift.** With shift source 1 and fourth argument 3, section C writes byte 6 at reduced selector + 7, then derives retained wide-shift value 0x00200000, then reads the selector byte and performs its selected write. F-C13 receives 0x00200000 three times. This order does not establish behavior for a shift source whose signed shift is unresolved.

3. **Changed selector.** If section C reads selector byte 5, its selected write targets 0x16. If section D subsequently reads 9, its zero write targets 0x1a and selector base becomes 18. Reusing the earlier 5 is inconsistent with the represented accesses.

4. **Payload widths.** Payload 0x12345678 yields logical right shift 0x02468acf. The saved payload byte is 0xcf, while section F’s full-word payload write contains 0x02468acf.

5. **Saved selection.** Saved selector 9 gives selector base 18 even if later dependencies alter the original selector byte. Selector-base accesses use the retained value.

6. **Reduced-selector bound.** Scale result 0xffffffff yields reduced selector 1023 and the fourth-argument-derived destination 1030. This establishes the extraction and destination only; it does not resolve other scale-derived signed interpretations or access validity.

7. **Equal target, different result treatment.** A static description assigning different numerical targets to F-A07 and F-E07 contradicts the supplied representation. Treating their represented result widths as automatically identical is also unsupported.

8. **Fresh final selection.** If the first final indexed read returns 2 and the second returns 7, their zero writes target 0x13 and 0x18 respectively.

9. **No normal success.** Reaching F-H18 does not establish successful return. The next represented endpoint is invalid-instruction termination.

## Uncertainties

### Established static evidence

The supplied representation establishes the described access widths, data dependencies, call order, masks, constants, numerical target equality, and abnormal endpoint. It places the section-C wide-shift calculation between the fourth-argument-derived write and the following selector read.

### Inferred intent

No application-level intent is adopted. Repeated offsets and scaling may suggest structured storage or encoded addressing, but they do not establish either interpretation.

### documented_only

Unresolved matters include:

- Correctness of the recovered instruction stream and function boundary.
- Whether additional entry values are genuine hidden inputs.
- Signed overflow, signed-shift behavior, and out-of-range unsigned-to-signed conversion.
- Machine address width and arithmetic beyond the represented model.
- ABI interpretation of paired results and the auxiliary register value.
- The result-width discrepancy at the shared F-A07/F-E07 numerical target.
- Alignment, endianness, atomicity, concurrency, and low-address validity.
- Dependency return behavior, memory effects, register preservation, and ownership effects.
- Reachability and external handling of the invalid-instruction endpoint.
- Any normal-return, resource-management, or error-reporting contract.

## Evidence

This contract uses only the supplied static representation. Its warnings identify invalid instruction data and truncated control flow, limiting conclusions about executable behavior.

No dependency implementation, validated machine-code interpretation, memory map, ABI documentation, or runtime trace was supplied. No runtime tests have been performed. Acceptance examples express conditional arithmetic and ordering consequences, not observed execution.
