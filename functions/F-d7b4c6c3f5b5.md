<!-- Trusted provenance: F-d7b4c6c3f5b5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7b4c6c3f5b5

## Purpose

The supplied static representation describes a context-dependent operation that stores an input word, invokes a sequence of unresolved dependencies, performs additional word and byte writes, and reaches an explicitly truncated control-flow boundary.

No application-level purpose is established. In particular, the evidence does not establish that this operation initializes an object, manages a buffer, implements a protocol, or intentionally accesses hardware registers. The low-address byte writes and repeated indirect calls must not be assigned such meanings without additional evidence.

This contract describes the visible execution prefix, conditional on the supplied representation accurately reflecting the executable behavior. It does not establish a complete, normally returning function.

## Interface

### Explicit input

The operation has one explicit input represented as a signed 32-bit integer. Its original 32-bit representation is stored before any dependency is invoked. A later value is formed by shifting that input left by three bit positions.

The signed interpretation creates a qualification: the representation alone does not establish language-level behavior for negative inputs or signed left-shift overflow. Where this contract gives wrapped arithmetic examples, those examples are conditional on a 32-bit machine-bitvector interpretation.

### Implicit entry state

Execution also depends on state not supplied through the explicit parameter:

- A context base address used for the first word write and first word read.
- An entry-supplied control address used to select the first dependency.
- An entry-supplied 32-bit payload subsequently written to a computed address.
- Memory mappings, access permissions, and the calling convention required by every dependency.

These are required inputs to the visible behavior, not established global variables or fields of a known public object.

### Output

No explicit return value or normal return is visible. Several dependencies produce values used later, but none is established as a return value of F-d7b4c6c3f5b5.

If every dependency returns and every memory access succeeds, the visible prefix reaches a bad-instruction/truncation boundary. The actual behavior beyond that boundary is unknown.

### Relevant memory layout

Only accessed offsets are established. No complete structure layout, object size, alignment guarantee, or allocation boundary is available.

| Address basis | Offset | Access width | Visible use |
|---|---:|---:|---|
| Entry context base | 0x24 | 32 bits | Store original explicit input |
| Entry context base | 0x74 | 32 bits | Load a retained address-like word |
| Retained address-like word | 0x24 | 32 bits | Store input shifted left by three |
| Retained address-like word | 0x74 | 32 bits | Load an argument for a dependency |
| Retained address-like word | 0x01 | 8 bits | Select a low-address byte destination |
| Four times a dependency-derived word | 0x14 | 32 bits | Store entry payload or a retained scaled value |
| A scaled dependency-derived address | 0x01 | 8 bits | Select another low-address byte destination |

The 8-bit selectors are zero-extended before adding 0x11. Consequently, their selected write destinations range from 0x11 through 0x110, inclusive. They are not established as offsets relative to an object base.

## Behavior

### Interpretation and ordering

The sequence below records observable accesses and dependency invocations. Each later event is conditional on all preceding events completing.

Dependency labels F-U01 through F-U40 are local opaque references to invocation sites. They do not assert known callee identities, distinct implementations, or published dependency mappings.

“Unspecified arguments” means that the representation supplies no explicit argument expressions for that invocation. It does not establish that the callee has no inputs.

For arithmetic descriptions:

- A word is 32 bits.
- A byte is 8 bits.
- Logical right shifts operate on unsigned words.
- Conversion to a stored byte retains the low eight bits.
- Under a 32-bit bitvector interpretation, left shifts, address scaling, and address addition retain the low 32 bits. Signed-expression qualifications remain documented in **Uncertainties**.

### Initial context update

1. Store the original explicit input as a 32-bit word at entry-context offset 0x24.
2. Read a 32-bit word from entry-context offset 0x74. Retain this value for subsequent addressing; later operations do not visibly reload it from the entry context.
3. Form the input shifted left by three.
4. Invoke F-U01 with four explicit words, in this order:
   - The retained address-like word.
   - That word shifted left by one.
   - The shifted input.
   - The retained word with bit 31 cleared, then logically shifted right by two.
5. Invoke F-U02, F-U03, F-U04, and F-U05, in that order, with unspecified arguments.
6. Store the shifted input as a 32-bit word at offset 0x24 from the retained address-like word.
7. Read the 32-bit word at offset 0x74 from that same retained value and supply it to F-U06.
8. Invoke F-U07, F-U08, and F-U09, in order, with unspecified arguments.

The first read occurs after the first write. If those addresses alias, the read observes the state resulting from that write, subject to the actual memory model. Likewise, the later offset-0x74 read occurs after the offset-0x24 write and after the preceding dependencies.

### First returned pair and indexed word write

9. Invoke F-U10. The visible representation consumes its result as two 32-bit halves of a 64-bit value.
10. Retain the low half shifted left by two as the first scaled result.
11. Store the entry-supplied 32-bit payload at the address obtained by multiplying the high half by four and adding 0x14.
12. Invoke F-U11 with:
    - The first scaled result.
    - The first scaled result again.
    - The first scaled result again.
    - The high half logically shifted right by three.
13. Invoke F-U12 and consume another result as low and high 32-bit halves.
14. Invoke F-U13 with:
    - The second result’s low half.
    - The second result’s high half.
    - The second result’s low half again.
    - The second result’s high half logically shifted right by three.
15. Invoke F-U14, F-U15, and F-U16, in order, with unspecified arguments.
16. Invoke F-U17 and consume its result as a 32-bit word.
17. Store the first scaled result at the address obtained by multiplying the F-U17 result by four and adding 0x14.
18. Invoke F-U18 with the F-U17 result multiplied by four.
19. Invoke F-U19, F-U20, and F-U21, in order, with unspecified arguments.

No checks on either result pair or the F-U17 result are visible before their use in arithmetic or addressing.

### Two selector-based byte writes

20. Invoke F-U22 and retain its unsigned 32-bit result.
21. Form a second scaled result by shifting that result left by two.
22. Read one unsigned byte at offset 1 from the retained address-like word.
23. Add 0x11 to that selector and store the low byte of the second scaled result at the resulting address.
24. Invoke F-U23 and F-U24, in order, with unspecified arguments.
25. Invoke F-U25 and retain its 32-bit result.
26. Read the selector byte at offset 1 from the retained address-like word again.
27. Add 0x11 to this newly read selector and store the low byte of the following expression at that address: the F-U22 result with its upper two bits cleared, logically shifted right by 20.
28. Invoke F-U26 with the F-U25 result shifted left by two.
29. Invoke F-U27, F-U28, and F-U29, in order, with unspecified arguments.
30. Invoke F-U30 and retain its 32-bit result.

The two selector reads are distinct observable accesses. They can produce different destinations because intervening dependencies, aliasing writes, or externally changing memory may alter the selector.

The first byte value contains bits 0–5 of the F-U22 result shifted into byte positions 2–7; its bottom two bits are zero. The second byte value contains original bits 20–27. Although the expression clears bits 30–31 before shifting, byte truncation also discards the shifted contributions from bits 28–29.

### Repeated zero writes through a second selector source

31. Read one unsigned byte at the address formed by multiplying the second scaled result by 0x400 and adding 1.
32. Add 0x11 to that selector and store a zero byte at the resulting address.
33. Invoke F-U31 with:
    - The second scaled result shifted left by three.
    - The same shifted value again.
    - The second scaled result without that additional shift.
    - The F-U30 result shifted left by two.
34. Invoke F-U32, F-U33, and F-U34, in order, with unspecified arguments.
35. Invoke F-U35 and retain its 32-bit result.
36. Read the selector byte from the same computed selector-source address again.
37. Add 0x11 to this newly read selector and store a zero byte at that address.
38. Invoke F-U36 with the F-U35 result shifted left by two.
39. Invoke F-U37, F-U38, F-U39, and F-U40, in order, with unspecified arguments.
40. Reach the visible bad-instruction/truncation boundary.

Under 32-bit bitvector arithmetic, the selector-source address in steps 31 and 36 is equivalently the F-U22 result shifted left by 12, plus 1. This equivalence does not establish that the address is mapped or valid.

## Side effects

The visible prefix contains:

- Four 32-bit writes.
- Four 8-bit writes.
- Two explicit 32-bit reads.
- Four explicit 8-bit selector reads.
- Forty dependency invocations.

These counts exclude all accesses performed inside dependencies, instruction fetching, and implicit calling-convention effects.

The writes are sequential, not transactional. No rollback, recovery, or cleanup path is visible. A fault, nonreturning dependency, or other interruption may leave any already-completed writes in place.

No allocation, release, ownership transfer, reference-count change, lock operation, or resource acquisition is explicitly established. Passing a value to a dependency does not prove transfer of ownership. Dependencies may independently allocate, release, mutate memory, or change global state; none of those possibilities can be resolved here.

No preservation guarantee is established for unrelated memory or globals. Conversely, this contract does not attribute unobserved side effects to a particular dependency.

The visible body does not check nullness, address range, alignment, overflow, dependency status, or selector validity. Required memory mappings include the computed low-address byte destinations, not merely the initial context memory.

## Dependencies

F-U01 is selected using an entry-supplied control address and a fixed displacement. Its target therefore depends on implicit entry state.

The remaining invocations use targets expressed through fixed address-derived calculations. Two invocation sites use a different displacement from most of the sequence. No authoritative function boundaries, signatures, or application-level meanings are available.

| Opaque invocation IDs | Explicit contract visible at the call site |
|---|---|
| F-U01 | Four word arguments; no consumed result |
| F-U06 | One word argument loaded immediately before invocation |
| F-U10, F-U12 | Results consumed as two 32-bit halves |
| F-U11, F-U13, F-U31 | Four explicit word arguments |
| F-U17, F-U25, F-U30, F-U35 | Results consumed as 32-bit words |
| F-U18, F-U26, F-U36 | One explicit scaled word argument |
| F-U22 | Result consumed as an unsigned 32-bit word |
| F-U02–F-U05, F-U07–F-U09, F-U14–F-U16, F-U19–F-U21, F-U23–F-U24, F-U27–F-U29, F-U32–F-U34, F-U37–F-U40 | Arguments unspecified; no result consumed by the visible representation |

No invocation has an established error convention. Results that are consumed are used directly, without a visible success test. Results not consumed cannot be interpreted as successful or failed operations.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Initial input storage and argument construction

Assume the explicit input is 3, the first offset-0x74 read produces 0x1000, and relevant accesses and dependencies complete.

Expected observations:

- The entry-context offset-0x24 write stores 3.
- F-U01 receives 0x1000, 0x2000, 24, and 0x400, in that order.
- After F-U02 through F-U05 return, the word write at address 0x1024 stores 24.
- The word at address 0x1074 is read afterward for F-U06.

This example does not assume dependencies preserve memory.

### Pair-result processing

Assume F-U10 produces low half 3 and high half 8.

Expected observations:

- The first scaled result is 12.
- The entry-supplied payload is written as a word at address 0x34.
- F-U11 receives 12, 12, 12, and 1.

If F-U17 subsequently returns 9, the first scaled result is written at address 0x38, and F-U18 receives 36.

### Byte extraction and selector rereading

Assume F-U22 returns 0x12345678.

Under 32-bit bitvector arithmetic:

- The second scaled result is 0x48D159E0.
- The first selector-based write stores byte 0xE0.
- The second selector-based write stores byte 0x23.

If the first selector read yields 0x20 and the second yields 0x21, the destinations are respectively 0x31 and 0x32. Reusing the first selector for both writes would not satisfy the visible access order.

### Repeated zeroing is not one operation

For the same F-U22 result, the second selector-source address is 0x45678001 under wrapped 32-bit arithmetic.

If its first read yields 5 and its later read yields 6:

- The first zero byte is written at 0x16 before F-U31.
- The second zero byte is written at 0x17 after F-U35.

Combining these into one write would lose observable behavior.

### Width-sensitive boundary case

Assume F-U22 returns 0x40000000 and machine arithmetic wraps at 32 bits.

The second scaled result becomes zero. Its selector-source address is therefore 1. Both extracted byte values are zero. There is no visible range guard preventing the read at address 1.

This criterion is conditional on bitvector semantics, not a claim that signed overflow is defined by a source language.

## Uncertainties

The following remain **documented_only**:

- **Instruction validity and completeness.** The supplied representation reports invalid instruction data and truncated control flow. Some displayed operations or call boundaries may not faithfully represent executable instructions.
- **Runtime purpose.** No domain-level intent is established.
- **Entry convention.** The origins and validity requirements of the implicit context base, control address, and payload are unknown.
- **Arithmetic semantics.** Word widths are explicit, but signed left-shift overflow and signed address-scaling behavior are not independently established. Wrapped examples are conditional.
- **Memory model.** Endianness, alignment requirements, mapped address ranges, device-memory semantics, and concurrent mutation rules are unknown.
- **Dependency behavior.** Signatures, preservation rules, ownership effects, exceptions, failures, and termination are unresolved.
- **Result representation.** Two results are represented as 64-bit quantities, but the actual calling convention for delivering their halves is unverified.
- **Final outcome.** The terminal marker does not establish a recoverable error, normal return, deliberate trap, or the behavior of any omitted continuation.

An inferred observation, not established intent, is that some intermediate values may be encoded addresses or packed data: they are repeatedly scaled and split. That pattern does not justify assigning a data format or object model.

## Evidence

The evidence consists solely of the supplied static function representation. It supports the listed access widths, offsets, expression dependencies, invocation order, repeated reads, and visible terminal boundary, subject to its instruction-validity warning.

No implementation access, independent instruction inspection, dependency inspection, or runtime tests have been performed. The acceptance examples are arithmetic and ordering criteria, not observed execution results. No snapshot identifier or independently verified provenance is asserted.
