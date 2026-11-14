<!-- Trusted provenance: F-cf6b1d3b92e1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cf6b1d3b92e1

## Purpose

The routine performs an ordered sequence of indirect dependency calls, 32-bit memory writes, byte reads, and byte writes. It propagates two explicit inputs, clears selected locations, stores scaled dependency results, and uses byte values as numeric destination selectors.

No application-level purpose is established. The offsets and multipliers do not establish a structure type, allocation layout, device protocol, or ownership model.

The supplied representation ends at an invalid-instruction boundary. It does not establish normal completion, a returned success indication, or recovery behavior.

This contract distinguishes:

- **Established static evidence:** represented calls, argument flow, memory-access widths, retained values, and operation order.
- **Conditional machine interpretation:** low-32-bit arithmetic used to explain computed values and addresses.
- **documented_only:** behavior not resolved by the supplied representation.

No runtime tests have been performed.

## Interface

### Explicit inputs and result

| Input | Width | Established direct use |
|---|---:|---|
| First input | 32 bits | None on the represented path. |
| Second input | 32 bits | Stored once at a dependency-selected address. |
| Third input | 32 bits | Stored at two computed addresses. |

No return value is represented. This is not evidence of a normal return: the visible continuation terminates at an invalid-instruction boundary.

The absence of a direct use of the first input does not establish that dependencies cannot observe it through machine state or an unresolved calling convention.

### Additional entry state

Execution also depends on three incoming machine-state values:

1. A target-related value used to select the first indirect call after adding `0xFFE`.
2. An unsigned 32-bit value passed to that call after a logical right shift by three.
3. An address factor used for the first third-input destination, multiplied by `0x80000` and increased by `0x24`.

Their initialization, validity, and relationship to the explicit interface are unknown. Reproducing only the three explicit inputs is insufficient to establish equivalent behavior.

### Widths and interpretation

A **word write** below means a represented 32-bit write. A **selector read** means an unsigned eight-bit read, yielding 0 through 255. A byte write retains the low eight bits of its value.

Address formulas and overflowing arithmetic use a **conditional 32-bit machine interpretation**: results retain their low 32 bits. This is consistent with the represented widths but is not independently validated as the execution semantics. Signed-looking multiplication and shifts must not be interpreted as portable signed-language arithmetic.

Logical right shifts of represented unsigned values are established. A represented 64-bit dependency result is explicitly divided into low and high 32-bit halves.

No endianness, alignment tolerance, atomicity, or valid address range is established. None of the numeric formulas implies an allocated object or pointer ownership.

## Behavior

### Dependency and ordering conventions

Opaque dependency identifiers **F-d01** through **F-d42** identify ordered call groups. They identify invocation sites, not proven distinct target functions. Multiple groups may reach the same target.

Within a group, calls occur in the stated order. “Four calls” means four successive invocations with no explicit arguments shown unless otherwise specified. It does not mean those calls are pure, argumentless under the actual ABI, or interchangeable.

A group’s **result** means its final call’s represented return unless specified otherwise. Results used in arithmetic are represented 32-bit values except for the explicitly identified 64-bit result. Intermediate results described as being forwarded have unresolved semantic types.

The sequence below specifies the represented direct effects between calls. Dependencies may produce additional effects.

### Initial input propagation and byte selection

The opening schedule is:

| Group | Invocation contract | Direct effect before the next group |
|---|---|---|
| F-d01 | One call using the incoming target-related state; receives the incoming unsigned value logically shifted right by three. | None represented. |
| F-d02 | Four calls. | Write the second input at `4 × result of F-d02 + 0x14`. |
| F-d03 | One call receiving `4 × result of F-d02`. | None represented. |
| F-d04 | Four calls. Its final unsigned result is called the **seed** below. | Write the third input at `0x80000 × incoming address factor + 0x24`. |
| F-d05 | One call receiving `16 × seed`, then `4 × seed`. | None represented. |
| F-d06 | Four calls. | Write the third input at `16 × seed + 0x24`. |
| F-d07 | One call receiving a word read from `16 × seed + 0x74`. | None after the call. |
| F-d08 | Four calls. | Perform the first byte-selected update described below. |

The word read supplying F-d07 occurs after the second third-input write and before F-d07 is invoked.

Define the retained **half-field** as:

`(seed AND 0x3FFFFFFF) logically shifted right by one`.

Its range is zero through `0x1FFFFFFF`.

After F-d08, read a selector byte at `16 × seed + 1`. Write a byte at the selector’s numeric value plus `0x11`, using the low eight bits of:

`(seed AND 0x3FFFFFFF) logically shifted right by 20`.

The destination therefore ranges from `0x11` through `0x110`; it is not an address loaded from a pointer-width field. The extracted value can contain ten bits before byte truncation.

The continuation is:

| Group | Invocation contract | Direct effect before the next group |
|---|---|---|
| F-d09 | One call receiving the result of F-d08 shifted left by two. | None represented. |
| F-d10 | One call producing a value. | None represented. |
| F-d11 | One call receiving that value. | None represented. |
| F-d12 | Four calls. | Re-read the selector at `16 × seed + 1`; clear the byte at its numeric value plus `0x11`. |
| F-d13 | One call receiving the result of F-d12 shifted left by two. | None represented. |
| F-d14 | Four calls. | Perform two ordered byte clears described below. |

After F-d14:

1. Clear the byte at `half-field + 0x0F`.
2. Read a selector at `1024 × half-field + 1`.
3. Clear the byte at that selector’s numeric value plus `0x11`.

The first clear is direct; it does **not** read a selector. It precedes the following selector read, including when their addresses overlap.

The selector read following F-d12 is fresh. Dependencies or earlier writes may have changed it since F-d08.

### Reset operations and retained selector

Define the **bank** as `1024 × half-field`, under the conditional 32-bit interpretation.

| Group | Invocation contract | Direct effect before the next group |
|---|---|---|
| F-d15 | One call receiving three copies of the result of F-d08 shifted left by 23, followed by that result shifted left by 25. | None represented. |
| F-d16 | One call producing a value. | None represented. |
| F-d17 | One call receiving that value. | None represented. |
| F-d18 | Four calls. | Read a selector at `bank + 1`, retain it, and clear the byte at its numeric value plus `0x11`. |
| F-d19 | One call receiving the result of F-d18 shifted left by two. | None represented. |
| F-d20 | Four calls. | Clear the word at the retained selector plus `0x44`, then clear the word at absolute address `0x24`. |
| F-d21 | One call receiving zero, `0x98`, bank, and `0x98`, in that order. | None represented. |
| F-d22 | Four calls: the first produces a value passed to the second; the third produces another value passed to the fourth. | None represented. |
| F-d23 | Four calls. | Clear the word at `4 × result of F-d23 + 0x44`, then at absolute address `0x24`. |
| F-d24 | One call receiving `4 × result of F-d23`. | None represented. |

The selector retained after F-d18 remains the operand for the word clear after F-d20. That clear does not re-read `bank + 1`, even if intervening dependencies modify it.

The next reset operations are:

| Group | Invocation contract | Direct effect before the next group |
|---|---|---|
| F-d25 | Four calls; the final result is represented as 64 bits. | None represented. |
| F-d26 | One call receiving the low 32 bits of F-d25’s result shifted left by two. | Clear the word at `2 × high32(result of F-d25) + 0x44`, then at absolute address `0x24`. |
| F-d27 | One call receiving zero, an additional result-register value associated with F-d26, and that value shifted left by two. | None represented. |
| F-d28 | One call with no explicit arguments shown, followed by one receiving `(half-field AND 0x3FFFFF) << 7`. | None represented. |
| F-d29 | Four calls. | Clear the word at `4 × result of F-d29 + 0x44`, then at absolute address `0x24`. |
| F-d30 | One call receiving `4 × result of F-d29`. | None represented. |

The high half of F-d25’s result is retained across F-d26. The extra register value passed to F-d27 has no established meaning, validity guarantee, or relationship to an ordinary return value.

There are four separate direct word clears at absolute address `0x24`: after F-d20, F-d23, F-d26, and F-d29. Calls and other writes separate them.

### Scaled-field writes

Define two retained numeric fields:

- **Scaled field:** `(half-field AND 0x3FFFFF) × 0x80`, ranging through `0x1FFFFF80`.
- **Small field:** `(half-field AND 0x3FFFFF) × 0x10`, ranging through `0x03FFFFF0`.

Neither calculation exceeds 32 unsigned bits.

| Group | Invocation contract | Direct effect before the next group |
|---|---|---|
| F-d31 | Four calls, then a fifth receiving the fourth result shifted left by two. | Clear the word at `4 × final result of F-d31 + 0x14`. |
| F-d32 | One call receiving the final result of F-d31 three times, followed by the scaled field. | None represented. |
| F-d33 | Four calls. | Clear the word at `4 × result of F-d33 + 0x14`. |
| F-d34 | One call receiving `4 × result of F-d33`, followed by four calls. | At `16 × result of F-d34 + 0x14`, write `4 × result of F-d34`. |
| F-d35 | One call receiving `4 × result of F-d34`, followed by four calls. | At `4 × result of F-d35 + 0x14`, write the retained `4 × result of F-d34`. |
| F-d36 | One call receiving `4 × result of F-d35`, followed by four calls. | At `16 × result of F-d36 + 0x14`, write `4 × result of F-d36`. |
| F-d37 | One call receiving `4 × result of F-d36` and `16 × result of F-d36`; one producing a value; one receiving that value and the small field; then four calls. | At `4 × result of F-d37 + 0x14`, write the scaled field. |

The F-d31 destination depends on the **fifth** call’s result, not merely the fourth call’s shifted result.

These stores establish numeric data flow only. In particular, the values scaled by four are not established pointers, sizes, or offsets.

### Final byte-selected updates and boundary

Define the **late base** as the low 32 bits of `half-field × 0x4000000`. Under the conditional machine interpretation, only the low six bits of the half-field affect this base.

| Group | Invocation contract | Direct effect before the next group |
|---|---|---|
| F-d38 | One call receiving `4 × result of F-d37`, followed by four calls yielding an unsigned result. | Read a selector at `late base + 1`; write the low byte of `4 × result of F-d38` at selector plus `0x11`. |
| F-d39 | One call receiving `4 × result of F-d38`; one producing a value; one receiving it; then four calls. | Re-read the selector at `late base + 1`; write the low byte of `(result of F-d38 AND 0x3FFFFFFF) >> 20` at selector plus `0x11`. |
| F-d40 | One call receiving the result of F-d39 shifted left by two, followed by four calls yielding an unsigned result. | Read and retain a selector at `late base + 1`; write the low byte of `4 × result of F-d40` at selector plus `0x11`. |
| F-d41 | One call receiving the retained selector zero-extended to 32 bits; one producing a value; one receiving it; then four calls. | Read a selector at `8 × result of F-d40 + 1`; write the low byte of `(result of F-d40 AND 0x3FFFFFFF) >> 20` at selector plus `0x11`. |
| F-d42 | One call receiving the result of F-d41 shifted left by two, followed by four calls. | No further direct memory action is represented. |

Each selector read is a separate operation. The final selector source depends on F-d40’s result, not on the late base.

After F-d42, the representation stops at an invalid-instruction boundary. A specific processor exception, normal return, or subsequent recovery path is not established.

## Side effects

If every represented direct operation is reached, the static inventory is:

| Direct operation | Count | Locations in the schedule |
|---|---:|---|
| Word writes using offset `0x14` | **7** | After F-d02, F-d31, F-d33, F-d34, F-d35, F-d36, and F-d37. |
| Third-input word writes using offset `0x24` | **2** | After F-d04 and F-d06. |
| Word clears using offset `0x44` | **4** | After F-d20, F-d23, F-d26, and F-d29. |
| Word clears at absolute address `0x24` | **4** | After those same four groups. |
| Byte writes | **9** | One after F-d08, one after F-d12, two after F-d14, one after F-d18, and one after each of F-d38 through F-d41. |
| Selector-byte reads | **8** | One after F-d08, F-d12, F-d14, F-d18, and each of F-d38 through F-d41. |
| Explicit word data reads | **1** | The argument read before F-d07. |

Thus there are **17 represented direct 32-bit writes**. These counts exclude dependency effects, instruction fetches, and any unrepresented machine-level accesses.

Computed addresses may alias one another, selector sources, or absolute address `0x24`. Ordering therefore matters; the contract does not promise an independent final value for every listed destination.

No allocation, release, ownership transfer, locking, or reference-count operation is directly established. Dependencies may nevertheless perform such operations. Direct access does not establish ownership of the affected memory.

No explicit bounds check, null check, error-status branch, rollback, or transaction is represented. Earlier effects can remain if a later access faults or a dependency does not return.

## Dependencies

All represented calls are indirect. Most targets are calculated from fixed call-site-associated values plus `0xFFE`; other represented displacements include `0xFC4`, `0xF1C`, `0xFEC`, `0xE02`, and `0xFF0`. F-d01 instead depends on incoming state.

These expressions do not establish executable targets, a stable ABI, or ordinary linkage.

No dependency is assumed to preserve globals, selector memory, other pointed-to memory, or additional result registers. No dependency is assumed to allocate, release, retain, validate, or transfer ownership. Calls need not return normally.

Traversal of the represented path requires valid reached targets, usable argument state, and valid memory accesses. The supplied material does not establish that these prerequisites hold.

## Acceptance examples

These are conditional static cases, not executed tests.

1. **Input propagation:** If F-d02’s result is 3 and the destination is valid, the second input is written as 32 bits at `0x20`. F-d03 receives `0x0C`.

2. **Independent third-input destinations:** With incoming address factor 2 and seed 3, the third input is written first at `0x100024` and later at `0x54`, with the intervening calls preserved.

3. **Byte selector, not pointer:** A selector value of `0xFF` after F-d08 selects destination `0x110`.

4. **Extraction truncation:** With seed `0x3FF00000`, the first extracted value is `0x3FF`; the stored byte is `0xFF`.

5. **Fresh selectors:** If a dependency changes the selector between F-d08 and F-d12, the later clear may target a different address. Reusing the earlier selector would not match the represented behavior.

6. **Direct clear before selector read:** If `half-field + 0x0F` aliases the selector source used immediately afterward, the clear after F-d14 precedes and can affect that read. No separate selector read belongs to the direct clear itself.

7. **Retained selector:** A retained selector of `0x20` after F-d18 makes the later word-clear address `0x64`, even if intervening calls change its original memory location.

8. **Wide-result retention:** If F-d25’s high half is 3, the word clear after F-d26 uses `0x4A`. Its low half supplies the shifted argument to F-d26 instead.

9. **Low-byte multiplication:** If F-d40’s result is `0x43`, its following byte write stores `0x0C`. F-d41 receives the retained selector, not `0x43` or `0x10C`.

10. **Repeated effects and partial execution:** Reaching all reset groups entails four distinct direct word clears at `0x24`. A dependency that stops returning can prevent later clears without requiring rollback of earlier ones.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended data structures.
- Correctness of the represented function boundary and call-target interpretation.
- Executability of the represented path.
- Exact ABI and provenance of additional entry state.
- Meaning of the additional result register and represented 64-bit return.
- Definitive overflow, address-width, alignment, endianness, and mapping behavior.
- Dependency effects, preservation rules, and ownership responsibilities.
- Concurrency, interrupts, atomicity, and memory-ordering guarantees.
- Execution beyond the invalid-instruction boundary.
- Any externally defined success or error protocol.

The supplied representation also reports overlapping global symbols. Accordingly, the four clears at `0x24` are specified as address-based effects, not as assignments to a validated independent global object.

## Evidence

This contract uses only the supplied static representation. The call schedule, argument relationships, access widths, selector re-reads, retained values, and direct-effect inventory are derived from its represented operations.

The arithmetic examples are conditional deductions, not runtime observations. Application-level interpretations are not established. No dependency implementation, execution trace, validated memory map, or ABI specification was available. No runtime tests have been performed.
