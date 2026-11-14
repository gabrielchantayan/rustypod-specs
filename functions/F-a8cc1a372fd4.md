<!-- Trusted provenance: F-a8cc1a372fd4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a8cc1a372fd4

## Purpose

The supplied representation describes a non-returning routine that repeatedly invokes unresolved dependencies, transforms their results, and performs interleaved byte and 32-bit memory accesses.

No application-level purpose is established. The repeated offsets, scaling operations, and indirect byte accesses may encode a structured state transformation, but the evidence does not justify identifying objects, queues, allocators, serialization formats, or ownership operations.

This contract describes the visible execution trace, conditional on the supplied control flow and operation widths being accurate. It does not establish that the representation is a complete, executable account of the original routine.

## Interface

**Explicit input:** one 32-bit value used initially as an address base.

**Additional entry state:** the first iteration also consumes:

- An incoming 32-bit machine value that supplies the first stored word.
- An incoming return-link-like value used to derive the first indirect call target.
- Addressable memory, executable indirect-call targets, and ambient state required by unresolved dependencies.

The supplied signature does not expose these additional inputs. Consequently, the explicit argument alone is insufficient to determine behavior.

**Output:** no ordinary return is visible. The loop has no visible exit condition. If execution continues normally, observable results consist of memory accesses, dependency invocations, and whatever effects those dependencies produce.

**Representation conventions used below:**

- A *word* is 32 bits; a *byte* is 8 bits.
- `word(address)` and `byte(address)` denote accesses of those widths.
- Arithmetic expressions describe 32-bit bit-vector transformations where applicable, retaining the low 32 bits. This is a conditional machine-level interpretation, not a claim that signed overflow in a higher-level language is defined.
- Unsigned right shifts insert zeroes. A stored byte retains only the low 8 bits.
- For a 64-bit dependency result, “low half” and “high half” mean bits 0–31 and 32–63.
- Descriptive value names below are specification labels, not source identifiers.
- Dependency IDs identify unresolved **call sites**, not proven distinct functions. A range denotes consecutive sites, invoked in ascending order. No-argument descriptions mean “no explicit arguments shown”; they do not establish absence of machine-state inputs.

No byte order, alignment policy, address-space mapping, or calling convention is established.

## Behavior

### Execution model and preconditions

The function performs its visible operations in the order below. Dependency calls are ordering boundaries: they must not be treated as pure, omitted, or moved across memory operations without additional evidence.

Normal continuation requires every reached call target to be executable and every reached memory access to be valid under the actual machine’s rules. No explicit validation, bounds checking, null checking, or recovery appears.

Some derived addresses can be extremely small. In particular, a byte read used as an address base is zero-extended to the range 0–255. Adding 17 therefore produces an address between 17 and 272. This is literal address arithmetic in the supplied evidence, not indexing through an independently established object base.

### Ordered iteration

The following trace covers one complete iteration. Intermediate values remain specification-level snapshots unless an explicit later read refreshes them.

1. **Initial store and first result.** Store the incoming carried word at the explicit input base plus 20. Invoke F-0101 with that base, then F-0102 through F-0105 without explicit arguments. Retain the 32-bit result of F-0105 as **First**. Store `First × 4` at address `First × 16 + 20`.

2. **First replicated argument.** Invoke F-0201 with three identical arguments, each `First × 4`. Invoke F-0202 through F-0205 without explicit arguments. Retain the final result as **Second**. Store `First × 4` at `Second × 4 + 20`.

3. **Second scaled record.** Invoke F-0301 with `Second × 4`, followed by F-0302 through F-0305 without explicit arguments. Retain the final result as **Third**. Store `Third × 4` at `Third × 16 + 20`. Establish **ByteBase** as `Third × 2³⁰`.

4. **Second replicated argument.** Invoke F-0401 with three copies of `Third × 4`, followed by F-0402 through F-0405 without explicit arguments. Retain the final result as **Fourth**. Store `Third × 4` at `Fourth × 4 + 20`.

5. **First byte clearing.** Invoke F-0501 with `Fourth × 4`, followed by F-0502 through F-0505 without explicit arguments. Retain the final unsigned result as **PackedA**. Write zero first to `ByteBase + 17`, then to `ByteBase + 1`.

6. **Short dependency sequence.** Invoke F-0601 with `PackedA << 2`. Invoke F-0602 without explicit arguments and pass its 32-bit result to F-0603. Invoke F-0604 through F-0607 without explicit arguments. Retain the final result as **Fifth**. Again write zero to `ByteBase + 17`, then to `ByteBase + 1`.

7. **Packed lookup.** Invoke F-0701 with `Fifth << 2`, followed by F-0702 through F-0705 without explicit arguments. Retain the final result as **Sixth** and establish **ScaledSixth** as `Sixth × 4`. Store `Fourth << 31` at `Sixth × 16 + 20`.

   Next read a word from `((PackedA & 0x3fffffff) >> 20) + 20`. Retain this loaded word as **LookupWord**. The address calculation spans 20–1043 and is not necessarily word-aligned.

8. **Retained result and word clearing.** Invoke F-0801 with ScaledSixth, then F-0802 without explicit arguments; retain the latter result as **Retained**. Invoke F-0803 with `(LookupWord >> 22) << 28`, followed by F-0804 through F-0807 without explicit arguments. Retain the final result as **Seventh**. Store a zero word at `Seventh × 4 + 20`.

9. **First indirect byte write.** Invoke F-0901 with `Seventh × 4`, followed by F-0902 through F-0905 without explicit arguments. Retain the final unsigned result as **PackedB**. Read a byte at `ByteBase + 1`; write the low byte of `PackedB << 2` at the loaded byte value plus 17.

10. **Second indirect byte write.** Invoke F-1001 with `PackedB << 2`. Invoke F-1002 without explicit arguments, then F-1003 with its result and `PackedB << 30`. Invoke F-1004 through F-1007 without explicit arguments; retain the final result as **Eighth**.

    Read a fresh byte at `ByteBase + 1` and retain it as **SavedByte**. Write the low byte of `(PackedB & 0x3fffffff) >> 20` at `SavedByte + 17`.

11. **Saved-byte word destination.** Invoke F-1101 with `Eighth << 2`, then F-1102 through F-1105 without explicit arguments. Store `Retained << 2` at `SavedByte + 20`. This destination uses the saved byte from step 10, not a fresh read.

    Read the visible ambient 32-bit global and retain its value as **GlobalSnapshot**.

12. **First split-width result.** Invoke F-1201 with ScaledSixth, followed by F-1202 through F-1205 without explicit arguments. Treat the final result as 64 bits. Establish **LowScaledA** as its low half shifted left two bits.

    Read a fresh byte at `ByteBase + 1`; write the low byte of `GlobalSnapshot >> 22` at that byte value plus 17.

13. **First indirect zeroing.** Invoke F-1301 with the high half of the preceding 64-bit result. Invoke F-1302 without explicit arguments, then F-1303 with its result and `LowScaledA << 28`. Invoke F-1304 through F-1307 without explicit arguments; retain the final result as **Ninth**.

    Read a fresh byte at `ByteBase + 1`; write zero at that byte value plus 17.

14. **First large-stride word store.** Invoke F-1401 with `Ninth << 2`, followed by F-1402 through F-1405 without explicit arguments. Retain the final result as **Tenth**. Invoke F-1501 with two copies of `Tenth << 2`, followed by F-1502 through F-1505 without explicit arguments. Retain the final result as **Eleventh**.

    Establish **WordA** as `Eleventh << 2`. Store WordA at `LowScaledA × 4096 + 20`.

15. **Mirrored word store.** Invoke F-1601 with `LowScaledA << 14`, followed by F-1602 through F-1605 without explicit arguments. Retain the final result as **Twelfth**. Store WordA at `Twelfth × 4 + 20`.

16. **Second split-width result.** Invoke F-1701 with `Twelfth × 4`, followed by F-1702 through F-1705 without explicit arguments. Treat the final result as 64 bits. Establish **LowScaledB** as its low half shifted left two bits.

    Read a byte at `LowScaledA × 16384 + 1`. At that byte value plus 17, write the low byte of `(WordA & 0x3fffffff) >> 20`.

17. **Additional split-width forwarding.** Invoke F-1801 with the high half of the preceding result. Invoke F-1802 without explicit arguments and pass its result to F-1803. Invoke F-1804 without explicit arguments, treating its result as 64 bits. Invoke F-1805 with that result’s low and high halves, in that order. Invoke F-1806 and F-1807 without explicit arguments; retain the final result as **Thirteenth**.

    Establish **HighScaledBase** as `LowScaledB × 2²⁸`. Read a byte at `HighScaledBase + 1`; write zero at the loaded byte value plus 17.

18. **Second byte-base selection.** Invoke F-1901 with `Thirteenth << 2`, followed by F-1902 through F-1905 without explicit arguments. Retain the final result as **Fourteenth**. Establish **WordBase** as `Fourteenth << 2`, and **IndirectBase** as `LowScaledB × 4`.

    Read a byte at `IndirectBase + 1`; write zero at the loaded byte value plus 17.

19. **Retained-value dependency argument.** Invoke F-2001 with `LowScaledB << 10`. Invoke F-2002 without explicit arguments, then F-2003 with its result and `Retained << 28`. Invoke F-2004 through F-2007 without explicit arguments; retain the final result as **Fifteenth**.

    Read a fresh byte at `IndirectBase + 1`; write zero at the loaded byte value plus 17.

20. **Second large-stride word store.** Invoke F-2101 with `Fifteenth << 2`, followed by F-2102 through F-2105 without explicit arguments. Establish **WordB** as `Fourteenth << 14`. Store WordB at `WordBase × 4096 + 20`.

    The result of F-2105 is not explicitly consumed by the visible subsequent arithmetic. Its effects and possible machine-state outputs remain unknown.

21. **Auxiliary-register argument.** Invoke F-2201 with `WordBase << 14`, then F-2202 without explicit arguments. Invoke F-2203 with `WordB << 28` and an auxiliary machine-register value not fully specified by the representation.

    Under 32-bit bit-vector arithmetic, the first argument is zero: WordB has already been shifted left 14 bits.

    Invoke F-2204 through F-2207 without explicit arguments; retain the final result as **Sixteenth**. Store WordB at `Sixteenth × 4 + 20`.

22. **Second packed-byte write.** Invoke F-2301 with `Sixteenth × 4`, followed by F-2302 through F-2305 without explicit arguments. Retain the final result as **Seventeenth** and establish **FinalBase** as `Seventeenth << 2`.

    Read a byte at `IndirectBase + 1`; write the low byte of `(WordB & 0x3fffffff) >> 20` at the loaded byte value plus 17.

23. **Final split-width forwarding.** Invoke F-2401 with `Fourteenth << 26`. Invoke F-2402 without explicit arguments, then F-2403 with its result and `FinalBase << 28`. Invoke F-2404 without explicit arguments, treating its result as 64 bits. Invoke F-2405 with the low and high halves, in that order. Invoke F-2406 and F-2407 without explicit arguments; retain the final result as **Eighteenth**.

    Read a fresh byte at `IndirectBase + 1`; write zero at the loaded byte value plus 17.

24. **Carry production and next iteration.** Invoke F-2501 with `Eighteenth << 2`, followed by F-2502 through F-2505 without explicit arguments. Retain the final result as **Nineteenth**. Establish **NextCarry** as `Nineteenth << 2`. Store NextCarry at `FinalBase × 4096 + 20`.

    Invoke F-2601 with four arguments, in order: `FinalBase × 1024`, the same value again, `Seventeenth << 14`, and NextCarry. Invoke F-2602 through F-2605 without explicit arguments.

    The final result, shifted left two bits, becomes the next iteration’s explicit base. NextCarry becomes the carried word for its initial store. The next iteration’s first call target is derived from the continuation state established at this iteration’s end, rather than the initial incoming link value.

The loop then repeats without a visible termination test.

## Side effects

### Memory and aliasing

Visible memory operations include:

- Word writes at the current iteration base plus 20.
- Word writes at result-derived addresses with multipliers 4, 16, or 4096.
- Direct byte zeroing at two offsets from ByteBase.
- Byte reads at offset 1 from several derived bases.
- Byte writes at a loaded byte value plus 17.
- A word write at a saved byte value plus 20.
- One packed-index word read and one ambient global read per completed iteration.

These locations are not established as disjoint. Writes may affect later reads, including reads that select subsequent write destinations. Repeated zero writes remain separate observable operations because intervening dependencies may change the selected byte, destination contents, or address mapping.

No read-modify-write atomicity, synchronization, or volatile-access guarantee is established.

### Ownership and lifetime

The visible body does not explicitly allocate or release memory. It does not establish ownership transfer for any argument or returned value.

Dependencies may allocate, release, mutate, retain, or invalidate memory. No preservation guarantee exists for the ambient global or memory reachable through derived addresses. A retained scalar snapshot does not imply continued validity of memory associated with that scalar.

### Errors and termination

No explicit error result, exception translation, rollback, or cleanup path appears. A dependency may fail to return or may transfer control elsewhere. Invalid memory or call targets may prevent continuation; the exact failure mechanism is unknown.

The visible infinite loop does not prove that every invocation runs forever, because faults and dependency behavior are unresolved.

## Dependencies

F-0101 through F-2605, including the seven-site ranges identified above, are opaque call-site IDs introduced solely for this contract. Their numbering does not encode addresses or assert callee identity.

The visible target expressions use several different derivation patterns. The first site depends on incoming machine state; most later sites derive targets from continuation-associated values and fixed displacements. No semantic classification of these targets is established.

The contract records explicit arguments and consumed results only. It does not assume:

- That a dependency with no explicit arguments has no inputs.
- That a discarded explicit result means the call has no effects.
- That separate sites reach separate functions.
- That dependencies preserve globals, memory, auxiliary registers, or ownership.
- That a nominal 32-bit or 64-bit result description establishes the actual ABI.

The auxiliary value passed at F-2203 is a specific unresolved input. It must not be replaced with zero or inferred from the preceding explicit result.

## Acceptance examples

These are conditional analytical examples, not executed tests.

1. **Initial store ordering.** Given a valid initial base and carried word, the word write at base plus 20 precedes F-0101. A trace that invokes F-0101 first does not match the visible order.

2. **Scaled result.** If First is 3 and arithmetic follows the stated 32-bit interpretation, step 1 writes 12 at address 68. Step 2 subsequently passes three copies of 12 to F-0201.

3. **Byte-address indirection.** If the step 9 byte read returns 200, its destination is address 217. If PackedB is 65, the stored byte is 4, because only the low byte of 260 is retained.

4. **Fresh versus saved selector.** If step 10 captures SavedByte as 7, the step 11 word destination is address 27 even if intervening dependencies change the selector byte in memory. Conversely, steps explicitly requiring fresh reads must observe the byte available at those later reads.

5. **Packed lookup boundary.** PackedA equal to zero selects address 20. PackedA equal to `0x3fffffff` selects address 1043. Acceptance depends on the target permitting the resulting word access; alignment is not guaranteed.

6. **High-bit truncation.** Under 32-bit arithmetic, `Third × 2³⁰` depends only on Third’s low two bits. It is not an unrestricted large-integer product.

7. **Repeated zeroing.** If an intervening dependency changes the byte at `IndirectBase + 1`, successive zero writes can target different addresses. Combining them into one store is not behaviorally justified.

8. **Loop-carried state.** If F-2605 returns 9, the next iteration starts with base 36. Its first word store uses NextCarry, not the original incoming carried word.

## Uncertainties

**Documented_only:**

- The accuracy of recovered control flow, indirect targets, and call signatures.
- The real calling convention and meanings of implicit register inputs.
- The initial carried word, incoming link value, and F-2203 auxiliary argument.
- Dependency identities, effects, failure modes, and memory ownership behavior.
- Whether low addresses are valid mapped storage, special-purpose locations, or evidence of an inaccurate representation.
- Alignment requirements, byte order, and behavior on invalid accesses.
- The interpretation of signed multiplication and left shifts beyond representable signed ranges.
- Whether the apparent 64-bit returns represent actual scalar returns or paired machine-register outputs.
- The semantic role of the ambient global and any overlapping storage.
- Whether hidden control flow changes the apparent non-returning behavior.

**Inferred intent:** the repeated scale/store/call patterns suggest a systematic transformation. No stronger application-level interpretation is supported.

## Evidence

Established static evidence consists of the supplied representation’s operation ordering, explicit access widths, constants, argument expressions, consumed result portions, repeated loop, and absence of a visible return path.

The representation also reports discarded unreachable material and overlapping global symbols. Those reports limit confidence in completeness and layout interpretation; they do not independently establish additional behavior.

No implementation access, dependency inspection, runtime execution, or tests have been performed. The acceptance examples are arithmetic and ordering consequences of the stated conditional interpretation, not runtime observations.
