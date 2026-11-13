<!-- Trusted provenance: F-e3a99d2f5a4f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3a99d2f5a4f

## Purpose

The supplied static representation describes a procedure that interleaves unresolved indirect calls with byte reads, byte writes, and 32-bit writes to computed addresses. Several writes clear storage; others store scaled call results or bit fields derived from incoming machine state.

No application-level purpose is established. In particular, the evidence does not justify identifying the procedure as an allocator, initializer, serializer, device driver, or collection operation.

The visible path ends at an invalid-instruction boundary rather than an ordinary return. This contract therefore describes the ordered, visible execution prefix, conditional on each preceding call returning and each memory access succeeding. It does not establish successful completion.

## Interface

### Explicit inputs

The displayed interface has four argument positions:

| Position | Displayed width | Visible use |
|---|---:|---|
| First | 32 bits | No direct use in the visible body. |
| Second | 32 bits | Supplied twice to the first indirect call. |
| Third | 32 bits | Supplied once to the first indirect call. |
| Fourth | 32 bits, displayed as signed | Shifted left by two positions and supplied to the first indirect call. |

These are raw words. Their interpretation as pointers, counts, handles, or flags is unknown.

### Additional incoming machine state

The visible computation also depends on values outside the four displayed arguments:

- **Incoming word R:** an unsigned 32-bit value used in address formation and logical right shifts.
- **Incoming word S:** an unsigned 32-bit value used to derive the first byte written.
- **Incoming link value:** used to form the first indirect-call target by adding `0xFC4`.

These names are descriptive labels for this contract, not additional established formal parameters. Their presence means the four-argument signature alone is insufficient to reproduce the visible behavior.

A later call also consumes a machine-register value exposed after an earlier call. Its meaning and relationship to that call’s ordinary return value are unknown.

### Outputs

No explicit return value is established. Observable outputs are:

- Ordered calls into unresolved dependencies.
- Explicit memory reads and writes.
- Any additional effects performed by those dependencies.
- An eventual invalid-instruction boundary on the displayed path, if execution reaches it.

The evidence does not establish an error code, exception contract, or normal return.

### Width and address conventions

The representation uses:

- Unsigned 8-bit reads.
- 8-bit stores.
- 32-bit word stores.
- Predominantly 32-bit arithmetic and address expressions.
- One call result represented as 64 bits, subsequently separated into low and high 32-bit halves.

For compactness, the behavior below uses these descriptive quantities:

- **Q:** R logically shifted right by three.
- **T:** Q multiplied by `0x80000`.
- **Low byte:** the least significant eight bits of a value.

The intended machine interpretation of scaling operations appears to be 32-bit arithmetic. However, signed multiplication and left shifts in the supplied representation do not establish portable overflow semantics. Wrapped numerical examples below are conditional on a 32-bit modular machine model.

## Behavior

### Execution and ordering rules

The following phases are ordered. Within each phase, calls and explicit memory operations occur in the listed order.

Dependency identifiers such as **F-A01** are opaque labels assigned here to unresolved call occurrences. They do not assert that two occurrences reach different functions, or that any target has a known implementation.

A call described as having “no displayed arguments” may still receive live machine-register contents. It is not evidence of a verified zero-argument ABI.

### 1. Initial calls and first byte update

1. **F-A01** receives the second explicit argument twice, then the third argument, then the fourth argument shifted left by two. Its target is formed from the incoming link value plus `0xFC4`.
2. **F-A02** produces a 32-bit word.
3. **F-A03** receives that word.
4. **F-A04**, **F-A05**, and **F-A06** run in order, with no displayed arguments.
5. **F-A07** produces a 32-bit result, retained as the first index.
6. One unsigned byte is read from address R plus one. Retain this byte as the first selector.
7. One byte is written at address first selector plus `0x11`. Its value is the low byte of S logically shifted right by 22.
8. **F-A08** receives the first index shifted left by two.
9. **F-A09**, **F-A10**, **F-A11**, and **F-A12** run in order.

The first selector is an actual byte value, not a pointer-sized value loaded from R. Consequently, the byte-write destination lies between `0x11` and `0x110`, inclusive.

### 2. First explicit clears and second index

1. A 32-bit zero is written at first selector plus `0x44`.
2. A 32-bit zero is written at absolute address `0x24`.
3. **F-B01** receives zero and `0x98`.
4. **F-B02** produces a word, passed to **F-B03**.
5. **F-B04** produces another word, passed to **F-B05**.
6. **F-B06**, **F-B07**, and **F-B08** run.
7. **F-B09** produces the second index.
8. A 32-bit zero is written at four times the second index plus `0x44`.
9. A 32-bit zero is written at address `0x24`.

The first selector remains the byte captured in phase 1. There is no intervening explicit reread of R plus one before its word-clear address is used.

### 3. Split-result processing

1. **F-C01** receives four times the second index.
2. **F-C02**, **F-C03**, and **F-C04** run.
3. **F-C05** produces a result displayed as 64 bits. Retain both 32-bit halves.
4. **F-C06** receives the low half shifted left by two.
5. A 32-bit zero is written at twice the high half plus `0x44`.
6. A 32-bit zero is written at address `0x24`.
7. **F-C07** receives three displayed arguments: zero, a register value exposed after F-C06, and that register value shifted left by two.
8. **F-C08** runs.
9. **F-C09** receives Q.
10. **F-C10**, **F-C11**, and **F-C12** run.
11. **F-C13** produces the third index.
12. A 32-bit zero is written at four times the third index plus `0x44`.
13. A 32-bit zero is written at address `0x24`.

The high half used in step 5 belongs to the earlier F-C05 result. It is not established as an output of F-C06.

### 4. First pair of `0x14` clears

1. **F-D01** receives four times the third index.
2. **F-D02**, **F-D03**, and **F-D04** run.
3. **F-D05** produces an index.
4. **F-D06** receives that index shifted left by two and produces a new 32-bit result.
5. A 32-bit zero is written at four times this new result plus `0x14`.
6. **F-D07** receives that new result in each of its first three displayed argument positions, followed by Q.
7. **F-D08**, **F-D09**, and **F-D10** run.
8. **F-D11** produces another index.
9. A 32-bit zero is written at four times this index plus `0x14`.

The two clear destinations are calculated from different call results. Their equality is possible but not required.

### 5. Retained scaled-value stores

1. **F-E01** receives four times the last index.
2. **F-E02**, **F-E03**, and **F-E04** run.
3. **F-E05** produces an index. Retain four times this index as the saved scaled value.
4. The saved scaled value is written as a 32-bit word at sixteen times that index plus `0x14`.
5. **F-E06** receives the saved scaled value.
6. **F-E07**, **F-E08**, and **F-E09** run.
7. **F-E10** produces a new index.
8. The previously saved scaled value is written at four times this new index plus `0x14`.

The value written in step 8 is derived from F-E05, not F-E10. Calls between the two stores may have their own memory effects; their implementations are unknown.

### 6. Further scaled store and Q store

1. **F-F01** receives four times the last index.
2. **F-F02**, **F-F03**, and **F-F04** run.
3. **F-F05** produces an index.
4. Four times this index is written at sixteen times the index plus `0x14`.
5. **F-F06** receives four times the index and sixteen times the index.
6. **F-F07** produces a word.
7. **F-F08** receives that word and R logically shifted right by six.
8. **F-F09**, **F-F10**, and **F-F11** run.
9. **F-F12** produces another index.
10. Q is written as a 32-bit word at four times this index plus `0x14`.

The two right shifts of R are distinct: the argument to F-F08 uses a six-bit shift, while the final store uses a three-bit shift.

### 7. First T-selected byte write

1. **F-G01** receives four times the last index.
2. **F-G02**, **F-G03**, and **F-G04** run.
3. **F-G05** produces an unsigned 32-bit word, retained as the first field word.
4. T is calculated from Q.
5. An unsigned byte is read from T plus one.
6. At that byte value plus `0x11`, one byte is written containing the low byte of the first field word shifted left by two.
7. **F-G06** receives the first field word shifted left by two.

The shifted call argument is a word-sized value; the memory store retains only eight bits.

### 8. Second T-selected byte write

1. **F-H01** produces a word.
2. **F-H02** receives that word.
3. **F-H03**, **F-H04**, and **F-H05** run.
4. **F-H06** produces an index.
5. A fresh unsigned byte is read from T plus one.
6. At that byte value plus `0x11`, one byte is written containing the low byte of the following extraction from the first field word: clear its top two bits, then shift right by 20.
7. **F-H07** receives the index from F-H06 shifted left by two.

The selector is reread after the intervening calls. It must not be assumed equal to the selector read in phase 7.

### 9. Third T-selected byte write

1. **F-I01**, **F-I02**, and **F-I03** run.
2. **F-I04** produces an unsigned 32-bit word, retained as the second field word.
3. A fresh unsigned byte is read from T plus one and retained as the final T selector.
4. At final T selector plus `0x11`, one byte is written containing four times the low byte of the second field word, truncated to eight bits.
5. **F-I05** receives the final T selector, zero-extended to a word.

This call receives the selector itself, not the byte written and not the write destination.

### 10. Final byte write and terminal boundary

1. **F-J01** produces a word.
2. **F-J02** receives that word.
3. **F-J03**, **F-J04**, and **F-J05** run.
4. **F-J06** produces an index.
5. An unsigned byte is read from eight times the second field word plus one.
6. At that byte value plus `0x11`, one byte is written containing the low byte of the second field word after clearing its top two bits and shifting right by 20.
7. **F-J07** receives the index from F-J06 shifted left by two.
8. **F-J08**, **F-J09**, **F-J10**, and **F-J11** run.
9. The visible representation reaches an invalid-instruction boundary.

There is no established ordinary epilogue after F-J11.

## Side effects

### Explicit memory footprint

If the complete visible prefix executes, it contains:

- Five explicit unsigned-byte reads.
- Five explicit byte writes.
- Four 32-bit zero writes to address `0x24`.
- Four additional 32-bit zero writes using a `0x44` displacement.
- Two 32-bit zero writes using a `0x14` displacement.
- Four nonzero-or-data-dependent 32-bit writes using a `0x14` displacement.

These counts exclude every dependency’s effects. A data-dependent write may also contain zero.

All byte-write destinations are calculated by zero-extending a selector byte and adding `0x11`. The word write using the first selector may be unaligned; other computed word accesses can also require alignment assumptions.

### Aliasing and visibility

No non-aliasing guarantees are established. Different calculated destinations may coincide, overlap, or refer to memory affected by dependencies. An earlier write or intervening call can therefore affect a later selector read.

The repeated clears of address `0x24` are separate ordered operations. They must not be treated as one redundant store when describing observable behavior: calls intervene and may read or modify that location.

### Ownership and failures

No ownership transfer, allocation, release, or lifetime extension is established. Passing a value to a dependency does not prove whether it is owned, borrowed, retained, or interpreted as an address.

There are no visible checks for:

- Invalid call targets.
- Unreadable selector addresses.
- Unwritable destinations.
- Alignment faults.
- Arithmetic overflow.
- Dependency failures.

A fault, nonreturning dependency, or other interruption can terminate execution before later phases. No rollback is visible.

## Dependencies

All F-A01 through F-J11 identifiers used above denote unresolved call occurrences, with the phase-specific ranges defined in the behavior section. Their argument and result relationships are the established dependency contract; their implementations are not supplied.

Except for the incoming-link-relative first call, target expressions are derived from fixed locations in the supplied representation. Target equality across occurrences has not been resolved. The occurrence labels therefore must not be interpreted as a recovered call graph.

For every dependency, the following remain unknown:

- Full ABI and hidden arguments.
- Memory reads and writes.
- Effects on shared state.
- Allocation or release behavior.
- Error signaling and whether it returns.
- Register preservation beyond what the supplied representation appears to rely upon.

The 64-bit result of F-C05 and the additional register value following F-C06 are particularly sensitive to ABI interpretation.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **First byte extraction.** If S is `0xABC00000`, all preceding calls return, and the selector read from R plus one is `0x20`, the first explicit byte write targets `0x31` and stores `0xAF`. The subsequent word clear based on the same captured selector targets `0x64`.

2. **Repeated selectors may change.** If the three reads from T plus one return `0x10`, `0x20`, and `0x30`, their respective byte-write destinations are `0x21`, `0x31`, and `0x41`. Reusing the first selector for all three would violate the visible read sequence.

3. **Retained scaled value.** If F-E05 returns 3 and F-E10 returns 7, the phase-5 stores write 12 at addresses `0x44` and `0x30`, respectively. The second store does not write 28.

4. **Low-byte truncation.** If the first field word is `0x12345678`, phase 7 writes `0xE0`, while phase 8 writes `0x23`. F-G06 receives the word-sized shifted value, not merely `0xE0`.

5. **Second field word.** If the second field word is `0x12345678`, phase 9 writes `0xE0`; phase 10 writes `0x23`. Under 32-bit modular address arithmetic, the final selector-read address is `0x91A2B3C1`.

6. **Split result.** If F-C05 has low half 3 and high half 5, F-C06 receives 12, and the following computed word clear targets `0x4E`. Whether that unaligned address is usable is platform-dependent.

7. **Interrupted execution.** If F-B03 does not return, none of the explicitly listed operations following that call are reached on this visible path. Earlier writes are not visibly undone.

## Uncertainties

### Established static evidence

The supplied representation establishes the displayed call order, argument expressions, result reuse, explicit access widths, selector rereads, stores, and terminal invalid-instruction boundary.

### Inferred intent

The repeated call groups and scaled stores suggest structured state manipulation. No stronger semantic interpretation is supported.

### documented_only

The following are intentionally unresolved:

- Whether the displayed instruction decoding and function boundary are correct.
- Whether the apparent straight-line path is executable in its entirety.
- The actual calling convention and dependency identities.
- Whether incoming machine-state values remain stable across calls as the representation assumes.
- Overflow, address wrapping, byte order, alignment rules, and memory mapping.
- Whether low absolute addresses represent ordinary memory, device state, aliases, or decoding artifacts.
- The exact runtime behavior at the invalid-instruction boundary.
- Any application-level success or failure condition.

These unknowns are not filled with assumed library behavior or conventional ownership rules.

## Evidence

This contract uses only the supplied static representation. That representation explicitly warns about invalid instruction data and overlapping global symbols, limiting confidence in recovered control flow, types, and storage interpretation.

No dependency bodies, executable environment, ABI documentation, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are arithmetic consequences of the visible expressions under their stated conditions, not evidence of observed execution.
