<!-- Trusted provenance: F-2ecbeaac6e28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ecbeaac6e28

## Purpose

F-2ecbeaac6e28 describes a stateful operation whose visible behavior combines indirect calls, computed byte and word accesses, propagation of dependency results, and extraction of selected bit fields.

No domain-level purpose is established. The supplied representation does not justify identifying the operation as allocation, initialization, serialization, device control, or object construction.

The visible path contains 93 indirect invocation sites and ends at an invalid-instruction boundary, not a demonstrated normal return. This contract covers that visible prefix. It does not establish that the apparent entry point, recovered control flow, or dependency interfaces are valid at runtime.

## Interface

### Explicit inputs

| Argument position | Represented width | Visible use |
|---|---:|---|
| First | 32 bits | Supplied to F-D001. |
| Second | 32 bits | No explicit use shown. |
| Third | 32 bits | No explicit use shown. |
| Fourth | Signed 32 bits | Shifted left by ten; the result is supplied twice to F-D008 and subsequently stored as a word. |

The apparent nonuse of the second and third arguments does not prove their irrelevance to an actual machine-level calling convention. Dependency signatures are unresolved.

### Implicit entry inputs

The operation additionally consumes entry state providing:

- An address for the initial byte read.
- A byte value for the initial byte write.
- A control value used to construct the first indirect target.

The visible body does not initialize these inputs or establish their provenance.

### Outputs and termination

No explicit return value or normal completion is established. Observable outputs are direct memory writes, dependency invocations and their operands, dependency effects, and any fault or nonreturn that interrupts execution.

No success flag, structured error result, rollback mechanism, or error-recovery branch is shown. The invalid-instruction boundary is a static limitation, not proof of a particular runtime exception.

### Widths and address terminology

The following are contract labels rather than implementation identifiers:

- **Initial selector:** the unsigned byte read at entry.
- **Scaled input:** the fourth argument shifted left by ten.
- **Pair lower** and **pair upper:** the low and high 32-bit portions of F-D009’s apparent 64-bit result.
- **Scaled index:** an index multiplied by four.
- **Ordinary field address:** four times an index, plus `0x14`.
- **Wide field address:** sixteen times an index, plus `0x14`.
- **Retained index:** the signed 32-bit result from F-D064.

A byte access has width eight bits; a word access has width 32 bits. Byte order, alignment requirements, pointer width, and integer-to-address interpretation are not established.

Unsigned 32-bit arithmetic retains the low 32 bits. This rule must not be generalized to signed expressions: several index multiplications, additions, and shifts are represented as signed operations. Overflow and negative-operand behavior remain unresolved. Assigning a signed multiplication result to an unsigned quantity does not establish wrapping behavior for an earlier signed overflow.

Unless otherwise qualified, arithmetic examples assume representable signed operations and valid reached accesses.

## Behavior

### Entry transfer and initial calls

Before any dependency call, the operation reads one unsigned byte at the entry address plus one. It then writes the entry-supplied byte to the numeric address equal to that selector plus `0x11`.

The destination therefore lies between `0x11` and `0x110`, inclusive. It is not the entry address plus a selector-dependent offset.

The initial selector is retained for F-D006; no intervening reread is shown.

The initial invocation sequence is:

| Dependency | Explicit operands or consumed result |
|---|---|
| F-D001 | First explicit argument. |
| F-D002–F-D004 | No explicit operands shown; results not visibly consumed. |
| F-D005 | Produces a signed 32-bit result. |
| F-D006 | Zero-extended initial selector, followed by the F-D005 result shifted left by two. |
| F-D007 | No explicit operands shown. |
| F-D008 | Scaled input in both operand positions. |
| F-D009 | Produces an apparent 64-bit result. |

The fourth argument is shifted after F-D007 returns and before F-D008 is invoked.

After F-D009 returns, its high and low halves become pair upper and pair lower. Pair upper is used as a signed 32-bit index. The operation stores the scaled input at its ordinary field address.

No bounds check, address validation, or object lookup is shown before this store.

### Initial propagation and lower-half extraction

F-D010 receives pair upper. F-D011–F-D013 follow without explicit operands. F-D014 supplies a signed 32-bit index.

The operation stores pair upper at that index’s ordinary field address. F-D015 receives the new index multiplied by four. F-D016–F-D019 then execute without explicit operands.

The next word store uses unsigned arithmetic:

- Destination: pair lower multiplied by `0x10000000`, plus `0x14`.
- Stored value: pair lower multiplied by `0x04000000`.

Both products are reduced to 32 bits. The destination depends only on pair lower’s low four bits; the stored value depends only on its low six bits.

F-D020 receives the same product used as the stored value.

F-D021–F-D023 follow without explicit operands. F-D024 supplies another signed 32-bit index. At its ordinary field address, the operation stores pair lower masked with `0x1fffffff`, then logically shifted right by nine.

This extraction selects original bits 9 through 28 into result bits 0 through 19. Its range is zero through `0x000fffff`.

F-D025 receives the F-D024 index multiplied by four. F-D026–F-D028 execute without explicit operands, followed by F-D029, which supplies the first wide-field index.

### Four wide-field stages

Ordinary and wide field addresses are distinct layouts. A store at sixteen times an index plus `0x14` does not generally initialize the word at four times that index plus `0x14`.

The following stages occur in order:

| Stage | Direct accesses and principal call |
|---|---|
| First wide-field stage | Store the F-D029 index multiplied by four at its wide field address. F-D030 receives an unsigned word read from its ordinary field shifted right by twelve, the scaled index, and another unsigned word read from its ordinary field shifted left by two. |
| First propagation | F-D031–F-D033 execute without explicit operands. F-D034 supplies an index. Store the F-D029 scaled index at the new ordinary field address. F-D035 receives the F-D034 index multiplied by four. |
| Second wide-field stage | F-D036–F-D038 execute without explicit operands. F-D039 supplies an index. Store its scaled index at its wide field address. F-D040 receives that scaled index twice, followed by pair lower shifted left by 26. |
| Second propagation | F-D041–F-D043 execute without explicit operands. F-D044 supplies an index. Store the F-D039 scaled index at the new ordinary field address. F-D045 receives the F-D044 index multiplied by four. |
| Third wide-field stage | F-D046–F-D048 execute without explicit operands. F-D049 supplies an index. Store its scaled index at its wide field address. F-D050 receives that scaled index. |
| Third propagation | F-D051–F-D053 execute without explicit operands. F-D054 supplies an index. Store the F-D049 scaled index at the new ordinary field address. F-D055 receives the F-D054 index multiplied by four. |
| Fourth wide-field stage | F-D056–F-D058 execute without explicit operands. F-D059 supplies an index. Store its scaled index at its wide field address, then capture an unsigned word from its ordinary field. F-D060 receives the scaled index. |

F-D030 has two separate word-read occurrences. Their relative evaluation order is not established. If memory can change between reads, its first and third operands need not derive from the same word value.

The fourth stage’s ordinary-field read occurs after its wide-field store and before F-D060. Its captured value, rather than a later reread, supplies the following extraction.

### Captured-word extraction and final propagation

F-D061–F-D063 execute without explicit operands. F-D064 supplies the retained index.

The retained index multiplied by four is represented as an unsigned 32-bit quantity after the multiplication. The operation stores the captured word logically shifted right by twelve at that scaled quantity plus `0x14`. F-D065 receives the scaled quantity.

F-D066–F-D068 follow without explicit operands. F-D069 supplies another signed 32-bit index. The operation stores that index multiplied by four at its wide field address.

F-D070 receives, in order:

1. The retained index’s scaled quantity masked with `0x3f`, then shifted left by fourteen.
2. The F-D069 index multiplied by four.
3. The retained index shifted left by thirty.

Where multiplication by four has a valid 32-bit interpretation, the first operand can set bits only in positions 16 through 19. Its maximum is `0x000f0000`.

Under a 32-bit bit-pattern interpretation, the third operand depends only on the retained index’s low two bits. That observation does not resolve the signed shift’s behavior for negative or overflowing cases.

F-D071–F-D073 execute without explicit operands. F-D074 supplies an index. The operation stores the F-D069 scaled index at the F-D074 index’s ordinary field address. F-D075 receives the F-D074 index multiplied by four.

### Final selector reads and byte writes

F-D076–F-D078 execute without explicit operands. F-D079 supplies an unsigned 32-bit result.

The operation reads a selector byte at the address formed from the retained index multiplied by `0x10000000`, plus one. This address expression uses the retained signed index; its full-domain overflow behavior is unresolved.

It writes the low eight bits of the F-D079 result shifted left by two to the selector’s numeric value plus `0x11`.

F-D080 receives:

1. The F-D079 result shifted left by two.
2. The same shifted result.
3. The F-D079 result shifted left by four.

The first two operands pass through a represented signed 32-bit intermediate. The unsigned shifts themselves retain 32 bits; conversion behavior outside the signed range is not independently established.

F-D080’s indirect target uses a different fixed adjustment from surrounding calls. Target equivalence must not be assumed.

F-D081 supplies a 32-bit value, passed to F-D082. F-D083–F-D085 then execute without explicit operands. F-D086 supplies a signed 32-bit result.

Only after those calls does the operation read the selector byte again from the same computed selector address. This is a new memory read, not reuse of the preceding selector.

At the newly selected numeric address plus `0x11`, it stores the low eight bits of the F-D079 result masked with `0x3fffffff` and logically shifted right by twenty. The stored byte selects original bits 20 through 27; original bits 28 and 29 disappear during byte truncation.

F-D087 receives the F-D086 result shifted left by two. F-D088–F-D090 follow without explicit operands. F-D091 supplies another signed 32-bit result.

F-D092 receives that result shifted left by two and the zero-extended selector from the second final selector read. F-D093 follows without explicit operands. The visible path then reaches the invalid-instruction boundary.

## Side effects

The visible body contains **17 direct store occurrences: three byte stores and fourteen 32-bit stores**.

The word-store accounting is:

| Word-store group | Count |
|---|---:|
| Scaled-input store and initial pair-upper propagation | 2 |
| Unsigned-product store | 1 |
| Masked pair-lower extraction | 1 |
| First four wide-field stores | 4 |
| Three intervening propagation stores | 3 |
| Captured-word extraction | 1 |
| Final wide-field store using F-D069 | 1 |
| Final propagation store using F-D074 | 1 |
| **Total word stores** | **14** |

The byte stores are the entry transfer and the two final selector-directed writes.

There are six explicit memory-read occurrences: three byte reads and three word reads. These counts exclude dependency effects and instruction fetches. They describe the complete visible prefix, not an unconditional promise that execution reaches every occurrence.

Writes and calls are interleaved and nontransactional. A later fault or nonreturn can leave earlier effects in place; no rollback is shown.

Aliasing is unrestricted by the evidence:

- Selector-directed byte destinations may overlap other accessed state.
- Dependencies may change later-read memory.
- The two final selectors may differ.
- Ordinary and wide field addresses may coincide in particular cases but are not generally equivalent.
- Distinct stores may target the same address without ceasing to be distinct store occurrences.

No allocation, release, ownership transfer, retention permission, synchronization, or global-preservation guarantee is established. In particular, absence of a visible release in this body does not establish that dependencies cannot release memory.

## Dependencies

F-D001 through F-D093 are opaque identifiers for successive indirect invocation sites. They are not proof of 93 distinct target functions.

F-D001’s target is derived from entry control state with an adjustment of `0xffe`. Later target constructions use fixed control values in the supplied representation. F-D080 uses `0xfec` rather than the surrounding `0xffe` adjustment.

“No explicit operands” means only that none are displayed. It does not establish a zero-argument machine interface or exclude implicit state inputs.

Visible consumed results are:

- Signed 32-bit results from F-D005, F-D014, F-D024, F-D029, F-D034, F-D039, F-D044, F-D049, F-D054, F-D059, F-D064, F-D069, F-D074, F-D086, and F-D091.
- An unsigned 32-bit result from F-D079.
- A 32-bit value without established signedness from F-D081.
- An apparent 64-bit result from F-D009, split into two 32-bit portions.

Other return values are not visibly consumed. Every dependency’s memory effects, global effects, ownership behavior, failure modes, and return behavior remain unknown.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes that execution reaches the described operations, intervening calls return, accesses succeed, and relevant arithmetic has the stated interpretation.

1. **Initial selector.** If the entry read yields `0x7f` and the supplied write byte is `0xa5`, the first store writes `0xa5` at numeric address `0x90`. F-D006 later receives `127` as its first operand, even if the original source byte has changed.

2. **Scaled input.** With fourth argument `3`, F-D008 receives `3072` twice. If pair upper is `5`, the following word store writes `3072` at address `0x28`.

3. **Unsigned product reduction.** With pair lower `0x10`, multiplication by `0x10000000` reduces to zero. The product store targets `0x14` and writes `0x40000000`; F-D020 receives `0x40000000`.

4. **Masked lower-half extraction.** With pair lower `0xffffffff`, the word stored after F-D024 is `0x000fffff`, not a sign-extended value.

5. **Different field layouts.** If F-D029 returns `2`, the wide-field store writes `8` at `0x34`. F-D030’s two word reads use `0x1c`. If both reads yield `0x12345000`, its first and third operands are `0x00012345` and `0x48d14000`.

6. **Captured value.** If the word captured before F-D060 is `0x12345000`, the extraction store after F-D064 writes `0x00012345`. Subsequent mutation of the original field does not itself change the retained value.

7. **Final wide-field and propagation stores.** If F-D069 returns `2` and F-D074 returns `3`, the final wide-field store writes `8` at `0x34`; the following propagation store writes `8` at `0x20`. Both are word stores and both contribute to the fourteen-word-store total.

8. **Distinct final selector reads.** If F-D079 returns `0x0ab00003`, the final byte values are `0x0c` and `0xab`. If the first selector read yields `2` and the second yields `9`, the destinations are `0x13` and `0x1a`. F-D092 receives `9` as its second operand.

9. **Interrupted execution.** If F-D040 does not return, subsequent accesses and calls are not required to occur. No visible mechanism undoes preceding effects.

## Uncertainties

The following remain **documented_only**:

- **Recovery validity:** the supplied representation reports invalid instruction data and truncated control flow.
- **Entry and calling conventions:** apparent arguments, implicit inputs, result widths, and indirect signatures may not describe the actual executable interface.
- **Signed arithmetic:** overflowing multiplication, addition, signed left shifts, negative operands, and out-of-range unsigned-to-signed conversion are not resolved across the full input domain.
- **Address model:** pointer width, address conversion, alignment, permissions, and byte order are unknown.
- **Dependency behavior:** memory or global mutation, allocation, release, exceptions, faults, and nonreturn are unconstrained.
- **Memory stability:** no concurrency exclusion, device-memory semantics, atomicity, or ordering barrier is established.
- **Termination:** neither normal return nor the actual runtime consequence of the invalid-instruction boundary is demonstrated.
- **Domain semantics and ownership:** encoded-state manipulation is plausible, but no object model or ownership contract is established.

Safe execution would require every reached call target to be executable and every reached access to be valid for its width. The evidence does not provide a complete input-validation rule that guarantees those conditions.

## Evidence

**Established static evidence:** the supplied body supports the invocation sequence, explicit operands, retained results, access widths, field offsets, masks, shifts, separate selector reads, fourteen word stores, three byte stores, six read occurrences, and terminal invalid-instruction boundary described above.

**Inferred intent:** repeated index scaling, propagation, and extraction suggest manipulation of encoded or indexed state. This interpretation adds no behavioral requirement.

**Unknown:** executable validity, actual dependency contracts, full-domain signed arithmetic, address-space semantics, domain purpose, and normal completion.

No runtime tests or executable validation have been performed.
