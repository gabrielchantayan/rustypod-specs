<!-- Trusted provenance: F-4bea37bfdb75; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4bea37bfdb75

## Purpose

F-4bea37bfdb75 describes a nonterminating sequence of indirect calls, computed-address word stores, and byte updates. Each completed iteration derives the explicit argument and two carried values for the next iteration.

The supplied static representation does not establish an application-level purpose. In particular, it does not establish that the computed addresses identify ordinary objects, that the calls perform allocation, or that the byte writes implement a recognizable data structure. Interpreting the operation as initialization, serialization, memory management, or dispatch would require additional evidence.

This contract records the visible sequence and its limits. It does not treat the recovered function boundary, call signatures, or apparent arithmetic types as independently verified machine behavior.

## Interface

**Explicit input:** one apparent 32-bit integer argument, called the **iteration argument** below. Its initial value is passed to the first indirect call.

**Additional entry state:** the first iteration also depends on three incoming values that are not initialized within the supplied body:

- A base used for the initial word store.
- A 32-bit value written by that store.
- A code-location value used to derive the first indirect-call target.

These are material inputs, not optional implementation details. The explicit argument alone is insufficient to reproduce execution.

**Output:** no ordinary return is visible. The body has an unconditional back edge. A dependency that does not return, a memory fault, an invalid call target, or another external interruption can prevent further progress; none is handled locally.

**Memory widths:**

- Word accesses shown here are 32 bits.
- Byte accesses are 8 bits.
- Several dependency results are represented as 64 bits and explicitly separated into low and high 32-bit halves.
- Byte reads used in address construction are unsigned: their values range from 0 through 255.

No byte order, alignment policy, address-space mapping, calling convention, or thread-safety guarantee is established.

For the arithmetic descriptions below, **W** denotes retaining the low 32 bits. Left shifts and multiplication describe the apparent fixed-width bit transformations, conditional on the underlying execution using 32-bit machine arithmetic. This is not a claim that every displayed signed arithmetic expression has portable language-level overflow semantics.

## Behavior

### Trace conventions

The following notation describes observable operations rather than executable pseudocode:

- **Word slot at x:** the 32-bit access at address W(x + 20).
- **Direct byte at x:** the 8-bit access at address W(x + 1) or W(x + 17), as specified.
- **Indexed byte through x:** first read the unsigned byte at W(x + 1), then write a byte at the address consisting of that byte value plus 17.
- **E(x):** the unsigned value obtained by masking x to its low 30 bits and shifting right by 20. Its range is 0–1023.
- **Byte truncation:** a stored byte retains only the low eight bits of its stated value.

The dependency IDs below identify unresolved **call sites**, not proven distinct functions. Within a numbered stage, calls have IDs F-U followed by the stage number and a two-digit position. For example, F-U0101 and F-U0105 designate the first and fifth call sites of stage 1. These IDs do not assert target identity.

A **five-call group** consists, in order, of:

1. A call with the listed explicit arguments.
2. Three calls with no explicit arguments shown.
3. A final call whose result is used where stated.

A **four-call group** has its listed first call, two calls with no explicit arguments shown, and a final result-producing call. Calls without explicit arguments may still depend on machine state. Every call remains an observable boundary; no preservation or purity is assumed.

All stages below are ordered. A later stage is reached only if preceding calls return and preceding accesses complete.

### Iteration prefix and first word updates

1. **Entry store and first result.** Write the carried 32-bit value to the word slot at the incoming carried base. Execute a five-call group, beginning with the iteration argument. Denote its final 32-bit result by **A**.

2. **First paired word update.** Write W(4A) to the word slot at W(16A). Execute a five-call group whose first call receives W(4A) three times. Denote the final result by **B**.

3. **Second word update.** Write W(4A) to the word slot at W(4B). Execute a five-call group beginning with W(4B). Denote the final result by **C**.

4. **Third word update.** Write W(4C) to the word slot at W(16C). Retain **H = W(C × 2³⁰)**. Execute a five-call group beginning with W(4C) three times. Denote the final result by **D**.

5. **Fourth word update.** Write W(4C) to the word slot at W(4D). Execute a five-call group beginning with W(4D). Denote the final unsigned result by **E**.

6. **First direct byte clearing.** Write zero at W(H + 17), then zero at W(H + 1). Call F-U0601 with W(4E). Call F-U0602 without explicit arguments and retain its 32-bit result.

7. **Forwarded result.** Execute a five-call group beginning with the result from F-U0602. Denote its final result by **G**.

8. **Repeated direct byte clearing.** Again write zero at W(H + 17), then zero at W(H + 1). Execute a five-call group beginning with W(4G). Denote its final result by **J**.

The repeated clears cannot be removed from the behavioral trace: intervening calls may have changed memory, and the addresses may be externally observable.

### First sampled word and byte-indexed updates

9. **Word store followed by word read.** Write W(D shifted left by 31) to the word slot at W(16J). Then read a 32-bit word from address E(E) + 20; call the sampled word **S**. Call F-U0901 with W(4J), then call F-U0902 without explicit arguments. Retain the latter result as **K**.

10. **Derived argument and zero word.** Execute a five-call group beginning with W((S shifted right unsigned by 22) shifted left by 28). Denote the final result by **L**. Write zero to the word slot at W(4L).

11. **First indexed byte update.** Execute a five-call group beginning with W(4L). Denote its final unsigned result by **M**. Read the byte at W(H + 1), then write the low byte of W(4M) at the read value plus 17. Call F-U1106 with W(4M), then call F-U1107 without explicit arguments and retain its result.

12. **Second indexed byte update.** Execute a five-call group beginning with the F-U1107 result and W(M shifted left by 30). Denote its final result by **N**. Read the byte at W(H + 1) again and retain that byte as **b**. Write the low byte of E(M) at b + 17.

13. **Saved byte as a word base.** Execute a five-call group beginning with W(4N); its final result is unused in the visible continuation. Write W(4K) as a 32-bit word at b + 20. This uses the saved byte b, not a new read of W(H + 1). Then read the apparent global 32-bit word and retain that sample as **T**.

14. **First wide result.** Execute a five-call group beginning with W(4J). Treat the final result as a 64-bit value, with halves **P-low** and **P-high**. Retain **Q = W(4 × P-low)**. Read the byte at W(H + 1) anew, then write the low byte of T shifted right unsigned by 22 at the read value plus 17.

15. **High-half forwarding and indexed clearing.** Call F-U1501 with P-high, then F-U1502 without explicit arguments. Execute a further five-call group beginning with the F-U1502 result and W(Q shifted left by 28). Denote its final result by **R**. Read the byte at W(H + 1), then write zero at the read value plus 17.

The global sample T is taken before stage 14’s calls. Its later use does not imply that the global remains unchanged.

### Second word-pair sequence

16. **Preparation for the next word store.** Execute a five-call group beginning with W(4R). Denote its final result by **V**.

17. **Duplicated argument.** Execute a five-call group beginning with W(4V) twice. Denote its final result by **Y**. Retain **Z = W(4Y)** and write Z to the word slot at W(Q × 4096).

18. **Matching word store.** Execute a five-call group beginning with W(Q shifted left by 14). Denote its final result by **AA**. Write Z to the word slot at W(4AA).

19. **Second wide result and indexed byte.** Execute a five-call group beginning with W(4AA). Split its final 64-bit result into **AB-low** and **AB-high**. Retain **AC = W(4 × AB-low)**. Read the byte at W(Q × 16384 + 1), then write the low byte of E(Z) at the read value plus 17.

20. **Wide-result forwarding.** Call F-U2001 with AB-high, then F-U2002 without explicit arguments. Call F-U2003 with that result. Call F-U2004 without explicit arguments and split its 64-bit result into low and high halves. Call F-U2005 with those halves, low first. Call F-U2006 without explicit arguments, then F-U2007 without explicit arguments and retain its result as **AD**. Read the byte at W(AC × 2²⁸ + 1), then write zero at the read value plus 17.

21. **Different indexed clearing.** Execute a five-call group beginning with W(4AD). Denote its final result by **AE**, and retain **AF = W(4AE)**. Read the byte at W(4AC + 1), then write zero at the read value plus 17.

### Third word-pair sequence

22. **Earlier result reused.** Call F-U2201 with W(AC shifted left by 10), then F-U2202 without explicit arguments. Execute a further five-call group beginning with the F-U2202 result and W(K shifted left by 28). Denote its final result by **AG**. Read the byte at W(4AC + 1) again, then write zero at the read value plus 17.

23. **Shift-derived word store.** Execute a five-call group beginning with W(4AG). Its final result is unused in the visible continuation. Retain **AH = W(AE shifted left by 14)**. Write AH to the word slot at W(AF × 4096).

24. **Additional machine-state argument.** Call F-U2401 with W(AF shifted left by 14), then F-U2402 without explicit arguments. Execute a further five-call group whose first call receives W(AH shifted left by 28) and an additional 32-bit machine-state value exposed after the preceding call. Denote the group’s final result by **AJ**. Write AH to the word slot at W(4AJ).

Under 32-bit truncating arithmetic, W(AH shifted left by 28) is zero because AH already has fourteen trailing zero bits. This does not make the call or its other argument dispensable.

25. **Indexed byte from the retained word.** Execute a five-call group beginning with W(4AJ). Denote its final result by **AK**, and retain **AL = W(4AK)**. Read the byte at W(4AC + 1), then write the low byte of E(AH) at the read value plus 17.

26. **Third wide-result forwarding.** Call F-U2601 with W(AE shifted left by 26), then F-U2602 without explicit arguments. Call F-U2603 with that result and W(AL shifted left by 28). Call F-U2604 without explicit arguments and split its 64-bit result into low and high halves. Call F-U2605 with those halves, low first. Call F-U2606 without explicit arguments, then F-U2607 without explicit arguments and retain its result as **AM**. Read the byte at W(4AC + 1), then write zero at the read value plus 17.

### Loop continuation

27. **Next carried value.** Execute a five-call group beginning with W(4AM). Denote its final result by **AN**. Retain **AP = W(4AN)**. Write AP to the word slot at W(AL × 4096).

28. **Next argument and base.** Execute a five-call group beginning with four arguments, in this order:

   - W(AL × 1024).
   - W(AL × 1024), repeated.
   - W(AK shifted left by 14).
   - AP.

   Retain the final result, scaled by four with 32-bit truncation, as the next iteration argument and next carried base. AP becomes the next carried store value. The next iteration’s first call target is derived from the code-location state established at this tail.

There is no completion check before the next iteration’s entry store.

## Side effects

The routine directly performs the stores and reads described above. Dependencies may perform additional accesses or mutate unrelated state.

Important constraints:

- **No ownership contract is established.** No visible operation proves allocation, acquisition, transfer, retention, or release.
- **No validation is visible.** Computed addresses are used without null checks, range checks, alignment checks, or bounds checks.
- **Byte-derived destinations are small addresses.** An unsigned byte plus 17 gives addresses 17–272; an unsigned byte plus 20 gives addresses 20–275. These are not automatically fields of H or another larger object.
- **Aliasing is permitted by the evidence.** Word stores, direct byte stores, indexed accesses, and the apparent global may overlap, depending on addresses and memory mapping.
- **Fresh reads matter.** Repeated byte reads separated by calls must not be replaced with a cached value. Conversely, stage 13 explicitly uses the previously saved byte.
- **No rollback is visible.** A failure after a completed store leaves that store in the trace; there is no local recovery path.
- **No synchronization is visible.** Concurrent observers, memory-mapped devices, interrupts, and dependency mutations could make repeated accesses observably different.

## Dependencies

All F-U identifiers above are document-local opaque call-site IDs. No external dependency identity can be established from the supplied dependency map.

Most call targets are derived from code-location values with a common displacement; several use different displacements. The first target additionally depends on incoming code-location state. Those differences must not be normalized into a single helper without further evidence.

For every unresolved dependency, the following remain documented_only:

- Actual signature and calling convention.
- Whether arguments or results are interpreted as pointers, integers, handles, or packed fields.
- Whether it returns normally.
- Whether it changes globals or accessed memory.
- Whether it owns, retains, allocates, or releases memory.
- Whether separate call sites resolve to the same implementation.
- Whether no-explicit-argument calls consume hidden state.

The wide-result descriptions preserve the supplied low/high-half usage; they do not prove a conventional 64-bit return ABI.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

1. **Initial ordering.** Given a valid initial store address and returning dependencies, the carried word is stored before F-U0101 receives the initial iteration argument.

2. **First paired stores.** If A is 3 and B is 5, stage 2 writes 12 at address 68. Stage 3 later writes 12 at address 40. Calls between these writes remain part of the trace.

3. **Unsigned byte indexing.** If the byte read in stage 12 is 255, the byte destination is address 272, and stage 13’s word destination is address 275. A signed-byte interpretation would violate the represented behavior.

4. **Saved versus refreshed byte.** If stage 12 reads 7 and a stage 13 dependency changes the byte at W(H + 1) to 9, stage 13’s word store still targets address 27. A later indexed operation that rereads that byte can instead target address 26.

5. **Byte truncation.** If E(M) is 1023, stage 12 writes byte value 255, not a multi-byte representation of 1023.

6. **Sampled-word address boundary.** For E equal to zero, stage 9 reads address 20. For E with all low thirty bits set, it reads address 1043. No scaling by four is shown for this address calculation.

7. **Bit-width simplification.** Under 32-bit arithmetic, stage 24’s first group argument is zero for every AE. Its second argument remains unresolved machine state.

8. **Continuation.** If stage 28’s final result is 6 and AP is 28, the next iteration begins by writing 28 at address 44, then passing 24 to its first call.

9. **Interrupted progress.** If a dependency never returns, no subsequent stage is required to occur. There is no visible local timeout or alternative branch.

## Uncertainties

The following are documented_only, rather than filled in by inferred intent:

- Whether the apparent function entry and unconditional loop correspond to the original executable’s intended boundaries.
- The origin and validity of the hidden entry values.
- Whether the displayed signed arithmetic is a faithful representation of fixed-width machine operations.
- The identity and layout of the apparent global word, including overlap with other symbols.
- Whether unusual low or unaligned addresses are valid mapped storage.
- The exact provenance of stage 24’s extra machine-state argument.
- Architectural fault behavior and dependency exception behavior.
- Whether dependencies preserve any global state or memory.
- Any application-level meaning of the stored values.

The available warnings about unreachable material and overlapping symbols reduce confidence in structural and type interpretations. They do not establish a replacement control flow or layout.

## Evidence

**Established static evidence:** the supplied representation shows one explicit integer input, hidden incoming state, ordered indirect calls, byte and 32-bit accesses, three low/high forwarding sequences, a sampled global word, and an unconditional iteration back edge. The stages above preserve the visible ordering of those operations.

**Inferred interpretation:** treating arithmetic as 32-bit truncating machine arithmetic, and treating the displayed wide results as low/high machine words, provides a coherent bit-level reading. These interpretations remain conditional where machine-level verification is absent.

**Unknown:** dependency semantics, valid memory mappings, ownership, fault handling, and application intent.

No runtime tests, executable traces, or implementation inspection beyond the supplied static representation have been performed.
