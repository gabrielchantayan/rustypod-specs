<!-- Trusted provenance: F-df86d32fbf38; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df86d32fbf38

## Purpose

The supplied static representation describes a sequence of indirect calls interleaved with byte and 32-bit memory accesses, arithmetic transformations, and one direct call to F-674e75a953e8.

No application-level purpose is established. The evidence does not justify describing this function as an allocator, serializer, device operation, or data-structure constructor. Repeated writes at offsets 0x11 and 0x14 suggest recurring storage conventions, but their meanings are unknown.

The representation explicitly reports invalid instruction data and truncated control flow. This contract therefore distinguishes:

- **Established static evidence:** operations and ordering visible in the supplied representation.
- **Conditional arithmetic consequences:** results under the stated 32-bit machine-arithmetic interpretation.
- **Unknown behavior:** dependency effects, calling conventions, memory validity, and execution beyond the truncated endpoint.

This is a source-facing contract, not a claim that the entire represented sequence can execute successfully.

## Interface

### Explicit inputs and output

The represented interface has three explicit inputs:

| Input | Represented width | Visible use |
|---|---:|---|
| First input | 32 bits | Passed to the first indirect call. |
| Second input | 32 bits | No explicit use is visible. |
| Third input | 32 bits, represented as signed | Shifted left by 26; the result participates in address calculations and a further multiplication. |

The represented return type supplies no result. No ordinary return is visible: the final indirect call is followed by an invalid-instruction termination marker.

Unused explicit input does not establish that the corresponding machine register is irrelevant to callees.

### Additional incoming state

Execution depends on state outside the explicit parameter list:

- An incoming register supplies the address of the first byte read.
- Another incoming register supplies the first byte written.
- The incoming link register participates in determining the first indirect call target.
- Three separately represented secondary-register outputs supply later values and addresses.

The extraction does not establish which calls define those secondary outputs, their complete lifetimes, or a reliable architectural ABI.

### Width and notation

The following notation describes values and access patterns without assigning application semantics:

- **wrap32(x):** the low 32 bits of a mathematical result.
- **byte(x):** the low eight bits.
- **B(a):** an unsigned, one-byte read at address a.
- **W(a):** a 32-bit read at address a.
- **Byte-slot write:** read B(a), zero-extend it, then write one byte at address B(a) + 0x11.
- **Word-slot write for index x:** write a 32-bit value at wrap32(4x + 0x14).

These are explanatory labels, not established object types.

All byte-slot destinations lie between 0x11 and 0x110 inclusive. A byte read is not a full-width pointer dereference: its value directly selects a low-address destination.

Unsigned right shifts are logical. Byte writes retain only eight bits. Products, address additions, and left shifts are described below using 32-bit bit patterns where applicable.

**Arithmetic qualification:** several operations are represented using signed integers. The text alone does not establish portable language-level behavior for signed overflow or invalid signed left shifts. Wraparound deductions are conditional machine-level interpretations, not unconditional language guarantees.

## Behavior

### Call-order conventions

The following ordered stages preserve visible memory-access and call ordering.

A **four-call chain** means four consecutive indirect calls with no explicit arguments shown; only the fourth call’s primary result is retained where specified. “No explicit arguments shown” does not establish that a callee reads no argument registers.

Other call groups are counted explicitly. Distinct indirect call sites must not be assumed to invoke one shared function.

### 1. Initial byte update and input-derived operations

1. Read one byte at the incoming address register plus 1. Retain this byte as the **initial selector**.
2. Write the incoming byte-register value at initial selector + 0x11.
3. Make the first indirect call, passing the first explicit input. Its target is formed from the incoming link register plus 0xFFE.
4. Execute a four-call chain; retain its final primary result as the **initial result**.
5. Form the **shifted input** by shifting the third explicit input left by 26.
6. Call an indirect dependency with initial result shifted left by 2 and the zero-extended initial selector.
7. Make one indirect call without explicit arguments.
8. Form the **scaled input** by multiplying shifted input by 0x400.
9. Read two 32-bit arguments from shifted input + 0x1C and pass them to an indirect call.
10. Make one further indirect call.
11. Perform a byte-slot write using scaled input + 1 as the selector-read address, writing zero.

The two reads in step 9 use the same address. Their relative evaluation order is not established, and they must not be collapsed into one read without a memory-model justification.

Under 32-bit wrapping arithmetic, scaled input is zero for every third input: the total left displacement is 36 bits. Consequently, step 11 then reads its selector at address 1.

### 2. First secondary-output sequence

Call an indirect dependency with three shown arguments: zero, the **first secondary value**, and scaled input. Execute a four-call chain, retaining its final primary result.

Next:

1. Re-read the selector byte at scaled input + 1.
2. Write byte(first secondary value shifted right by 22) to that selector’s byte slot.
3. Call an indirect dependency with the retained primary result shifted left by 2.
4. Execute another four-call chain, retaining its final primary result.
5. Call an indirect dependency with scaled input shifted left by 26 and that result shifted left by 2.
6. Make one indirect call and retain its result as the **retained word**.
7. Call F-674e75a953e8 with retained word in both shown argument positions.
8. Make one indirect call.
9. Store retained word at the word slot indexed by the **second secondary value**.

The retained word is the result obtained before F-674e75a953e8; the direct dependency’s result is not assigned to it.

### 3. Linked word and byte updates

Call an indirect dependency with the second secondary value and its low 30 bits shifted right by 10. Execute a four-call chain and retain its final result as an index.

Then:

1. Store the second secondary value at that index’s word slot.
2. Call an indirect dependency with four times that index.
3. Execute a four-call chain; retain the final unsigned result as the **first packed value**.
4. Read a selector byte at wrap32(8 × retained word + 1).
5. Write byte(4 × first packed value) to the selected byte slot.
6. Call an indirect dependency with four times first packed value.
7. Execute a four-call chain and retain its final result.
8. Read and retain a selector at wrap32(64 × first packed value + 1).
9. Write byte((first packed value masked with 0x3FFFFFFF) shifted right by 20) to that selector’s byte slot.
10. Call an indirect dependency with the latest chain result shifted left by 2.
11. Execute a four-call chain.
12. Call an indirect dependency with its final result shifted left by 2 and the retained selector.
13. Make one indirect call.
14. Call an indirect dependency with retained word shifted left by 10 in both shown argument positions.
15. Make one indirect call.
16. Read a selector at address 1 and write zero to its byte slot.

The last selector read follows from an explicitly zero-valued intermediate in the representation; it does not require the earlier scaled-input simplification.

### 4. Third secondary-output sequence

Call an indirect dependency with the **third secondary value** twice, followed by sixteen times that value. Execute a four-call chain and retain its final result.

Next:

1. Read and retain a selector at wrap32(16 × third secondary value + 1).
2. Write byte(third secondary value shifted right by 22) to its byte slot.
3. Call an indirect dependency with the retained chain result shifted left by 2.
4. Execute a four-call chain.
5. Call an indirect dependency with the retained selector and that chain’s final result shifted left by 2.
6. Make one indirect call.
7. Call an indirect dependency with zero in both shown argument positions.
8. Make one indirect call whose represented result is 64 bits.

Split that result into a **lower half** and an **upper half**, each 32 bits. This is a split of the represented result value, not a claim about memory byte order.

Then:

1. Store zero at the word slot indexed by the upper half.
2. Call an indirect dependency with the upper half.
3. Execute a four-call chain and retain its final result.
4. Store the upper half at that result’s word slot.
5. Call an indirect dependency with four times that result.
6. Make four indirect calls without explicit arguments.
7. Write wrap32(lower half × 0x04000000) at address wrap32(lower half × 0x10000000 + 0x14).

The third-secondary selector and the earlier selectors are not interchangeable, even if a particular execution gives them equal values.

### 5. Masked-word output and repeated scaled-index updates

Call an indirect dependency with wrap32(lower half × 0x04000000). Execute a four-call chain, retaining its final result as an index.

Store the lower half masked with 0x1FFFFFFF and shifted right by 9 at that index’s word slot. Call an indirect dependency with four times the index.

The following stages each begin with a four-call chain:

| Stage | Operations following the chain |
|---|---|
| First scaled-index stage | Let the final result be an index. Form a scaled index equal to four times it. Store the scaled index at wrap32(16 × index + 0x14). Read two words from scaled index + 0x14. Call an indirect dependency with one read shifted right by 12, the scaled index, and the other read shifted left by 2. |
| First link stage | Store the preceding scaled index at the final result’s word slot. Call an indirect dependency with four times that result. |
| Second scaled-index stage | Form a new scaled index from four times the final result. Store it at wrap32(16 × result + 0x14). Call an indirect dependency with that scaled index twice and lower half shifted left by 26. |
| Second link stage | Store the preceding scaled index at the final result’s word slot. Call an indirect dependency with four times that result. |
| Third scaled-index stage | Form another scaled index. Store it at wrap32(16 × result + 0x14). Call an indirect dependency with the scaled index. |
| Third link stage | Store the preceding scaled index at the final result’s word slot. Call an indirect dependency with four times that result. |
| Fourth scaled-index stage | Form another scaled index and store it at wrap32(16 × result + 0x14). Read and retain W(scaled index + 0x14). Call an indirect dependency with the scaled index. |

Each chain occurs after the preceding stage’s final call.

The two reads in the first scaled-index stage are separately represented reads. Their relative argument-evaluation order is unknown. The store at 16 × index + 0x14 is not generally a store at scaled index + 0x14; these addresses must not be conflated.

### 6. Final word and byte updates

Execute a four-call chain after the last call above. Let its final result be the **final field index**, and form the **field base** as four times that index.

1. Store the retained word from the fourth scaled-index stage, logically shifted right by 12, at field base + 0x14.
2. Call an indirect dependency with field base.
3. Execute a four-call chain. Form a **new base** as four times its final result.
4. Store new base at wrap32(16 × that result + 0x14).
5. Call an indirect dependency with:
   - field base masked with 0x3F, then shifted left by 14;
   - new base;
   - final field index shifted left by 30.
6. Execute a four-call chain.
7. Store new base at that chain’s final result’s word slot.
8. Call an indirect dependency with four times that result.
9. Execute a four-call chain, retaining its final unsigned result as the **final packed value**.
10. Read a selector at wrap32(final field index × 0x10000000 + 1).
11. Write byte(final packed value shifted left by 2) to its byte slot.
12. Call an indirect dependency with final packed value shifted left by 2 twice, followed by final packed value shifted left by 4.

The target formation for step 12 uses an offset of 0xFEC rather than the otherwise prevalent 0xFFE.

Make one indirect call and pass its primary result to the next indirect call. Then execute a four-call chain.

Finally:

1. Re-read the selector at wrap32(final field index × 0x10000000 + 1).
2. Write byte((final packed value masked with 0x3FFFFFFF) shifted right by 20) to its byte slot.
3. Call an indirect dependency with the last chain result shifted left by 2.
4. Execute a four-call chain.
5. Call an indirect dependency with that chain’s final result shifted left by 2 and the newly retained selector.
6. Make one final indirect call.

The representation terminates at an invalid-instruction marker. No subsequent behavior is established.

## Side effects

### Memory and ordering

Explicit effects include one-byte reads, one-byte writes, 32-bit reads, and 32-bit writes. No bounds, null, alignment, or permission checks are visible.

Writes are incremental. Earlier writes precede later calls and accesses; no rollback is shown. A later failure does not establish restoration of prior state.

Dependencies may alter any memory they can access. Consequently:

- A selector re-read after calls may differ from its earlier value.
- Repeated reads at one address may return different values.
- A previously stored word may be changed before a later read.
- Aliasing may make a byte write affect a subsequent selector read or word access.

The representation supplies access order, not synchronization or atomicity guarantees.

### Ownership and errors

No explicit allocation, release, ownership transfer, or resource-lifetime operation is established. Passing a word to a dependency does not establish that it is a pointer or that ownership transfers.

There is no visible error return, status check, recovery branch, or cleanup path. Invalid addresses, invalid targets, dependency failures, and the final invalid-instruction endpoint have no established application-level handling.

## Dependencies

**F-674e75a953e8** is the sole dependency with an established opaque function ID. It is called once, with the retained word in both shown argument positions.

All other calls are unresolved indirect dependencies. No opaque function IDs are supplied for their targets; none are invented here. Their argument lists and ordering are described above, but their internal contracts are documented_only.

No dependency is assumed to:

- preserve globals or reachable memory;
- preserve unmodeled registers;
- allocate or release memory;
- return normally;
- obey a particular ownership convention.

Most indirect targets are expressed relative to code-position-derived values using 0xFFE. The first uses incoming link state, and one late call uses 0xFEC. These expressions establish target-formation differences, not verified executable destinations.

## Acceptance examples

These are conditional review examples, not executed tests.

1. **Initial selector boundary.** If the first selector read returns 0xFF and the incoming byte value is 0xA6, the first explicit write stores 0xA6 at address 0x110.

2. **Input collapse.** Under wrapping 32-bit arithmetic, every third input produces scaled input zero. Its shifted form can still affect the earlier reads at shifted input + 0x1C.

3. **Byte truncation.** If first packed value is 0x41, its multiplication by four is 0x104. The associated byte write stores 0x04, not a 32-bit value.

4. **Masked extraction.** If first packed value is 0xFFFFFFFF, masking and shifting right by 20 yields 0x3FF. The subsequent byte write stores 0xFF.

5. **Selector mutation.** If an intervening dependency changes a selector byte, a later explicit re-read must use the changed selector. Reusing the earlier selector would not preserve the represented behavior.

6. **Split-result stores.** If the represented 64-bit result has upper half 3 and lower half 2, the first upper-half-indexed write stores zero at 0x20. The later lower-half-derived write stores 0x08000000 at 0x20000014, assuming execution reaches it and both addresses are writable.

7. **No fabricated success.** Reaching the final indirect call does not establish successful return. The visible endpoint is invalid instruction data, not a success result.

## Uncertainties

The following remain **documented_only**:

- Correctness of instruction recovery and completeness of control flow.
- Executability of the represented call sequence.
- Application purpose and interpretation of recurring offsets.
- Exact ABI, secondary-result provenance, and register preservation.
- Memory map, alignment requirements, byte order, and accessibility of low addresses.
- Dependency identities other than F-674e75a953e8.
- Dependency side effects, ownership behavior, and failure modes.
- Signed-arithmetic semantics where the representation uses signed operations.
- Relative evaluation order of same-address argument reads.
- Whether the invalid-instruction endpoint corresponds to an actual trap or extraction failure.

Apparent structure is not sufficient evidence to resolve these unknowns.

## Evidence

This contract is based solely on the supplied static representation: represented widths, explicit accesses, arithmetic, call arguments, call order, the direct dependency mapping, and the invalid-instruction warnings.

No implementation inspection beyond that supplied material, runtime tests, execution traces, or dependency implementations were available. Arithmetic examples are conditional deductions, not observed execution results.
