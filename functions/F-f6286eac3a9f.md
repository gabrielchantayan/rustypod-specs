<!-- Trusted provenance: F-f6286eac3a9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f6286eac3a9f

## Purpose

F-f6286eac3a9f repeatedly performs an ordered sequence of indirect calls, memory reads, memory writes, and a software breakpoint. If every call returns, every memory access succeeds, and execution resumes after the breakpoint, another pass begins unconditionally.

The static evidence establishes this sequence, not its application-level purpose. It does not establish whether the accessed locations represent ordinary objects, hardware registers, tables, or execution-context storage. Initialization, dispatch, and device-control interpretations remain unproven.

No normal return, finite iteration count, success result, explicit error result, or cleanup path is visible.

## Interface

### Inputs and outputs

There are no declared explicit arguments and no visible returned value. Execution nevertheless consumes implicit machine state:

- An incoming link value used to construct the first indirect-call target.
- A register-derived **context base** used for several memory accesses and explicitly adjusted during each pass.
- An incoming **auxiliary register value** contributing to a late byte-store address.
- Two global words used as address bases.
- Memory reached through fixed addresses, loaded pointers, and dependency results.
- Additional register outputs associated with three dependency calls.

Successful continuation requires the relevant call targets and memory locations to support their respective operations when reached. No visible checks enforce these requirements.

### Widths and arithmetic

The representation distinguishes:

| Quantity | Width |
|---|---:|
| Byte access | 8 bits |
| Halfword access | 16 bits |
| Word access and address-like value | 32 bits |
| Paired dependency result | 64 bits, divided into low and high 32-bit words |

Three paired results are named **first pair**, **second pair**, and **final pair**, according to when they are obtained.

Unsigned halfword loads are zero-extended when used in word-sized expressions. Narrow stores retain only the destination-width bits. A 32-bit store of a zero-extended halfword writes the complete word, including zero upper bits.

For explicitly unsigned word shifts, right shift is logical and left shift discards bits beyond the word width. Several other arithmetic expressions are represented with signed types. Their apparent machine-level behavior is described using 32-bit quantities, but portable source-language behavior for signed overflow and signed left shifts is not established. Acceptance examples involving those operations are conditional on the apparent 32-bit interpretation.

Byte order, alignment requirements, access permissions, atomicity, and volatile or memory-mapped status are unknown.

### Captured values

A value described as captured is the value obtained at its stated read or call-return point. Later dependencies may modify its original memory location; the visible continuation still uses the captured value unless a reload is specified.

This does not establish that dependencies preserve physical registers, referenced storage, or object lifetimes. Such calling-convention and lifetime questions remain unresolved.

## Behavior

F-unknown-01 through F-unknown-37 identify call sites in execution order. They are opaque dependency references, not assertions that the sites resolve to distinct implementations.

All operations following a call are conditional on that call returning to the displayed continuation.

### 1. Opening calls and byte stores

At the beginning of each pass, a byte value is established as zero for a later store.

1. Invoke **F-unknown-01** with two explicit zero arguments. Its target is formed by adding `0xffe` to the current link value. On entry to the first pass, that link value is an implicit input. Near the end of a completed pass, it is explicitly replaced.
2. Invoke **F-unknown-02**, then **F-unknown-03**. No explicit arguments are shown.
3. Read the **first global base** as a 32-bit word.
4. Invoke **F-unknown-04**, then **F-unknown-05**. Capture the latter’s 64-bit result as the first pair.
5. Invoke **F-unknown-06** with two word-sized arguments:
   - The first pair’s low word.
   - The first pair’s high word plus its low word plus a byte known to be zero.

   The second argument is a word-sized sum, not a carry-propagating 64-bit addition.
6. Store an 8-bit additional register output associated with F-unknown-06 at first-global-base offset `0x11`.
7. Invoke **F-unknown-07**.
8. Store zero as a byte at first-global-base offset `0x15`.
9. Invoke **F-unknown-08**. Store its 16-bit result at offset `0x0c` from an address supplied through an additional output register.

Both stores through the first global base use the earlier captured value. The global word is not visibly reloaded between them.

### 2. Second-pair processing and the breakpoint

10. Invoke **F-unknown-09**, capturing the second pair.
11. Write a zero halfword at second-pair-high-word plus `0x2c`.
12. Write the second pair’s unsigned low word shifted right by 26 as a word at context-base offset `0x26c`. The stored value is between 0 and 63.
13. Invoke **F-unknown-10**.
14. Read a word-sized address from context-base offset `0x320`, equivalent to decimal offset 800. Retain it as the **early destination**.
15. Invoke **F-unknown-11**, then **F-unknown-12**.
16. Write the second pair’s low word shifted left by two as a word at early-destination offset `0x54`.
17. Invoke **F-unknown-13**, **F-unknown-14**, **F-unknown-15**, **F-unknown-16**, and **F-unknown-17**, in that order.
18. Read an unsigned halfword from fixed address `0x21c48994`. Retain its zero-extended value as the **small base**.
19. Invoke **F-unknown-18**.
20. Read a word from context-base offset `0x280`. Pass that word to **F-unknown-19** and capture its word-sized result.
21. Execute a software breakpoint with immediate `0x92`.

The breakpoint is unconditional on the visible path. The evidence does not establish that execution resumes automatically. All remaining operations require resumption.

### 3. Post-breakpoint indexing and halfword stores

22. Shift the F-unknown-19 result left by two, forming a word-sized **index displacement**.
23. Derive an arithmetic base by logically shifting `0x21c48960` right by 18 and multiplying by four. Its value is `0x21c4`.
24. Read a word-sized address at small-base offset `0x44`. Retain it as the **later destination**.
25. Invoke **F-unknown-20**, then **F-unknown-21**. Capture the latter’s word-sized result.
26. Read a word at small-base offset `0x04`. Invoke **F-unknown-22** with the captured F-unknown-21 result as its first argument and this newly loaded word as its second.
27. Read the **second global base** as a word.
28. Invoke **F-unknown-23**.
29. At second-global-base plus index-displacement, store the low 16 bits of the captured second global base.

   This stores a truncated address-like value. It does not visibly read a halfword from the second global base and copy that halfword elsewhere.
30. Invoke **F-unknown-24**.
31. Read a word from arithmetic-base offset `0x54`, which is fixed address `0x2218`. Retain that word as a **selector source address**.
32. Invoke **F-unknown-25**, obtaining an address-like word result.
33. Store halfword `0x21c4` at that returned address plus `0x04`.
34. Invoke **F-unknown-26**, then **F-unknown-27**.
35. Read an unsigned halfword at selector-source-address offset `0x34`. Retain its zero-extended value as the **table selector**.
36. Invoke **F-unknown-28**, capturing the final pair.

The small base and table selector are separate halfword-derived values. The former supplies address bases earlier in this phase; the latter controls later absolute arithmetic addresses.

### 4. Final-pair processing and repetition

37. Store the final pair’s low word as a word at context-base offset `0x258`, equivalent to decimal offset 600.
38. Read an unsigned halfword at final-pair-high-word plus `0x14`. Retain its zero-extended value as the **field value**.
39. Form these word-sized quantities:
   - Final-pair-low-word shifted left by six.
   - Final-pair-high-word multiplied by four.
   - Table-selector multiplied by four.
40. Adjust the represented context base downward by `0xf4`, or 244 bytes.
41. Store the final pair’s low word at address table-selector-times-four plus `0x74`.

   No context base or separate table base is visibly added to this destination.
42. Invoke **F-unknown-29**.
43. Store the field value as a word at later-destination offset `0x54`.
44. Invoke **F-unknown-30**.
45. Store the same captured field value as a word at later-destination offset `0x64`, equivalent to decimal offset 100.
46. Invoke **F-unknown-31**.
47. Store the low 16 bits of the later destination at the address formed by adding:
   - Final-pair-low-word shifted left by six.
   - An additional register output associated with F-unknown-31.
48. Invoke **F-unknown-32**, then **F-unknown-33**.
49. Read a byte at address table-selector-times-five. This is the selector plus the previously formed selector-times-four quantity; no separate base is visibly included.
50. Invoke **F-unknown-34**.
51. Store the captured byte at field-value plus auxiliary-register-value plus `0x19`.

   The byte read precedes F-unknown-34, while the destination is used afterward. The evidence does not resolve whether that dependency can change the auxiliary register’s effective value.
52. Invoke **F-unknown-35**.
53. Store final-pair-high-word times four as a word at later-destination offset `0x44`.
54. Invoke **F-unknown-36**.
55. Replace the link value with `0x08712031`.
56. Read a word at final-pair-high-word-times-four plus `0x44`. Pass this loaded word to **F-unknown-37**.
57. If F-unknown-37 returns, begin another pass unconditionally.

The explicit link replacement would make the next opening target `0x0871302f` under the apparent address arithmetic. Dependency effects on machine state remain an independent uncertainty.

No explicit restoration of the context-base decrement appears before repetition. Whether this represents a genuine repeated physical execution-state adjustment is unresolved.

## Side effects

Each fully completed pass visibly contains:

- **37 indirect calls.**
- **One software breakpoint.**
- **Three byte stores.**
- **Five halfword stores.**
- **Seven word stores.**
- Reads from globals, fixed addresses, context-relative locations, and indirectly obtained addresses.
- One explicit downward context-base adjustment.
- One explicit link-value replacement.

The seven direct word stores are exactly:

| Order | Destination | Stored value |
|---|---|---|
| 1 | Context base plus `0x26c` | Second-pair low word shifted right by 26 |
| 2 | Early destination plus `0x54` | Second-pair low word shifted left by two |
| 3 | Context base plus `0x258` | Final-pair low word |
| 4 | Table-selector-times-four plus `0x74` | Final-pair low word |
| 5 | Later destination plus `0x54` | Zero-extended field value |
| 6 | Later destination plus `0x64` | Same captured field value |
| 7 | Later destination plus `0x44` | Final-pair high word times four |

There is no eighth direct word store in the supplied representation. These counts exclude dependency effects.

Writes occur in the stated order. No transaction, rollback, synchronization, memory barrier, or atomic grouping is established. Aliasing can make a later access observe or overwrite an earlier effect. A fault, nonreturning call, or nonresuming breakpoint can leave only a prefix of the sequence completed.

### Ownership and errors

No direct operation establishes allocation, ownership acquisition, transfer, or release. Loaded and returned addresses are used without visible validation.

Dependencies may change memory, invalidate addresses, or allocate or release storage. None of those behaviors is established or excluded. Retaining an address across a call does not prove its storage remains live.

There are no visible checks for null addresses, alignment, bounds, arithmetic overflow, dependency status, breakpoint support, or lifetime validity. No explicit error handler or recovery path appears. Machine faults and external interruption behavior are outside the established contract.

## Dependencies

All 37 call sites are indirect. The opening target depends on the current link value; other targets are formed from fixed address arithmetic in the representation.

The following data relationships are visible:

| Dependency | Explicit input or consumed output |
|---|---|
| F-unknown-01 | Two zero-valued arguments |
| F-unknown-05 | First 64-bit pair |
| F-unknown-06 | Two word-sized arguments; additional byte output consumed |
| F-unknown-08 | Halfword result and additional address output consumed |
| F-unknown-09 | Second 64-bit pair |
| F-unknown-19 | Loaded word argument; word-sized result consumed |
| F-unknown-21 | Word-sized result consumed |
| F-unknown-22 | Two word-sized arguments |
| F-unknown-25 | Address-like result consumed |
| F-unknown-28 | Final 64-bit pair |
| F-unknown-31 | Additional register output contributes to a store address |
| F-unknown-37 | Loaded word argument |

For the remaining sites, no explicit arguments or consumed ordinary results are shown. This is not evidence that they lack implicit inputs, register outputs, memory effects, or control-flow effects.

No dependency is assumed to preserve globals, preserve memory, maintain lifetimes, or own or release any referenced storage.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes all preceding operations complete and required memory is accessible.

1. **Word-sized first-pair sum.** If the first pair has low word `0xffffffff` and high word `0x00000002`, the apparent 32-bit interpretation gives F-unknown-06 arguments `0xffffffff` and `0x00000001`, not a widened second argument of `0x100000001`. Exact signed-overflow semantics remain unresolved.

2. **Unsigned second-pair shifts.** If the second pair’s low word is `0xffffffff`, the word written at context offset `0x26c` is `0x0000003f`. The later word written at early-destination offset `0x54` is `0xfffffffc`.

3. **Captured first global.** If the first global base is read as A and a later dependency changes that global word to B, the two early byte stores still use captured base A in the visible dataflow. This does not establish A’s continued validity.

4. **Truncated second-global value.** If the captured second global base is `0x12345678` and the index displacement is `0x10`, the indexed halfword store writes `0x5678` at `0x12345688`. It does not copy memory contents from `0x12345678`.

5. **Zero-extended field writes.** If the field halfword is `0xfedc`, both later-destination offsets `0x54` and `0x64` receive word `0x0000fedc`. F-unknown-30 executes between these writes. Its effects on the first destination are not visibly undone by the second write.

6. **Selector addresses.** For table selector 3, the final-pair low-word store targets address `0x80`, and the later byte read targets address `0x0f`. Neither expression visibly includes a separate table base.

7. **Breakpoint suspension.** If the breakpoint does not resume, no post-breakpoint access or call occurs. Earlier effects remain outside any visible rollback mechanism.

8. **Store inventory.** A fully completed pass executes seven direct word-store operations, in the order listed under Side effects. Aliased destinations do not increase or decrease that operation count.

9. **Unconditional repetition.** If every call returns, every access succeeds, and the breakpoint resumes, execution reaches another pass rather than a normal return.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and field meanings.
- Dependency identities, call-site aliasing, calling conventions, and implicit inputs.
- Reliability of the paired-result grouping and additional register outputs.
- Dependency effects on globals, registers, memory, and object lifetimes.
- Whether the context-base adjustment represents a genuine repeated physical state change.
- Signed-overflow and signed-shift interpretation beyond the apparent 32-bit operations.
- Storage ownership, byte order, permissions, alignment, atomicity, and volatility.
- Whether fixed and low numeric addresses are ordinary memory, hardware mappings, or representational artifacts.
- Breakpoint handling, resumption policy, faults, and external recovery.

A repeated low-level state-update sequence is plausible inferred intent, not an established functional purpose.

## Evidence

This contract derives solely from the supplied static representation. Established evidence covers visible call order, explicit call dataflow, memory-access widths and ordering, arithmetic relationships, breakpoint placement, explicit state adjustments, and unconditional repetition.

The direct word-store inventory contains seven operations: two before the breakpoint and five afterward.

No dependency bodies, runtime traces, implementation access beyond the supplied representation, or external documentation were available. No runtime tests have been performed. Acceptance examples express conditional consequences of the static evidence, not observed execution results.
