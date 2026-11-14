<!-- Trusted provenance: F-d7a0f1da7fee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7a0f1da7fee

## Purpose

The supplied static representation describes a long, straight-line sequence of indirect calls, address calculations, byte accesses, and 32-bit writes. It forwards the second explicit input to an initial dependency, stores the third explicit input at two calculated addresses, and repeatedly changes fields selected by dependency results and incoming machine state.

A higher-level purpose—such as initialization, object construction, serialization, or device configuration—is **not established**. The repeated field offsets and shifted values suggest structured state manipulation, but that is inferred intent, not an interface guarantee.

This contract describes the visible prefix only. The supplied representation ends at an invalid-instruction boundary rather than a normal return and carries a warning about unreliable instruction decoding. Consequently, the ordering below is established as an ordering in the supplied static representation, not as verified executable behavior.

## Interface

### Explicit inputs

The displayed interface has three 32-bit inputs:

- **Input 1:** No visible use in the supplied prefix.
- **Input 2:** Passed unchanged to the first indirect call.
- **Input 3:** Written unchanged as a 32-bit quantity at two calculated addresses.

These are bit-pattern descriptions. The representation does not establish that any input is a pointer, handle, signed quantity, or owned resource.

### Additional incoming state

The visible behavior also depends on three incoming machine values outside the displayed argument list:

- **Incoming dispatch base:** Used to calculate the first indirect call target by adding `0xFC6`.
- **Incoming shift value:** Interpreted as an unsigned 32-bit value and shifted right by three before the first visible word write.
- **Incoming region selector:** Multiplied by `0x80000`, then increased by `0x24`, to locate the first destination for Input 3.

Their presence prevents the three explicit arguments from being a complete calling contract. How callers establish these values is documented_only.

### Output and termination

No normal return value is established. The displayed interface has no declared result, and the visible prefix ends at an invalid-instruction termination marker.

Dependency return values nevertheless drive later addresses, writes, and calls. One dependency produces a displayed 64-bit result whose low and high 32-bit halves are used separately. Another contributes an auxiliary machine-register result in addition to its ordinary return behavior.

No success status, failure status, exception contract, or recovery path is visible.

### Arithmetic and access notation

For the descriptions below:

- **Word** means 32 bits; **byte** means 8 bits.
- **Word at an address** means a 32-bit memory access, not an array index.
- Address additions, products, and left shifts are described as 32-bit machine bit-pattern calculations, retaining the low 32 bits.
- Right shifts of explicitly unsigned quantities are logical.
- A byte read used as an address component is zero-extended and therefore lies between 0 and 255.
- A byte write retains only the low eight bits of its value.

The representation uses signed intermediates in several address calculations. Exact executable overflow behavior is not independently established; the low-32-bit interpretation is the intended machine-level reading, not a claim about portable signed arithmetic.

No byte order or alignment policy is supplied.

## Behavior

### Dependency notation

Each identifier of the form **F-I01.1** below is an opaque identifier for an unresolved indirect-call occurrence. Different identifiers do not establish different underlying functions; repeated-looking call patterns do not establish equal targets.

A “four-call run” means four consecutive indirect calls with no explicit arguments shown. Where its final result is named, the first three ordinary results are not visibly consumed. “No explicit arguments” does not prove that machine registers or ambient state are irrelevant.

Every step below is conditional on execution reaching it and all preceding calls returning.

### 1. Initial forwarding and first word update

1. **F-I01.1** receives Input 2.
2. **F-I01.2** produces a 32-bit result, which **F-I01.3** receives unchanged.
3. A four-call run, **F-I01.4–F-I01.7**, produces a final result called **A**.
4. The word at address \(4A + \texttt{0x14}\) is written with the incoming shift value logically shifted right by three.
5. **F-I01.8** receives \(4A\).

No direct use of Input 1 appears.

### 2. Establishment and propagation of a scaled value

A four-call run, **F-I02.1–F-I02.4**, produces **B**.

- Define **B4** as \(4B\).
- Write B4 to the word at \(16B + \texttt{0x14}\).
- Read the word at \(B4 + \texttt{0x14}\), logically shift that word right by three, and pass the result to **F-I02.5**.

This is a real read after the preceding write. The two addresses are not assumed equal.

A further four-call run, **F-I02.6–F-I02.9**, produces **C**.

- Write the retained B4 value to the word at \(4C + \texttt{0x14}\).
- Pass \(4C\) to **F-I02.10**.

### 3. Input 3 storage and persistent derived values

A four-call run, **F-I03.1–F-I03.4**, produces unsigned word **D**.

The following occur in order:

1. Write Input 3 to the word at \( \text{incoming region selector} \times \texttt{0x80000} + \texttt{0x24}\).
2. Define **D16** as \(16D\).
3. Define **H** as \((D \mathbin{\&} \texttt{0x3FFFFFFF}) \gg 1\).
4. Call **F-I03.5** with D16 and \(D \ll 2\).
5. Execute four calls, **F-I03.6–F-I03.9**.
6. Write Input 3 to the word at \(D16 + \texttt{0x24}\).
7. Read the word at \(D16 + \texttt{0x74}\) and pass it to **F-I03.10**.

The two writes of Input 3 are separated by five dependency calls. They are not an atomic pair, and their addresses may alias.

### 4. First byte-selected writes

A four-call run, **F-I04.1–F-I04.4**, produces **E**.

- Read one byte at \(D16 + 1\).
- Add `0x11` to that byte value to obtain a destination address.
- Write the low byte of \((D \mathbin{\&} \texttt{0x3FFFFFFF}) \gg 20\) there.
- Call **F-I04.5** with \(E \ll 2\).
- Call **F-I04.6** and forward its ordinary result to **F-I04.7**.
- Execute a four-call run, **F-I04.8–F-I04.11**, producing **G**.
- Read the byte at \(D16 + 1\) again.
- Write zero to the byte at that newly read value plus `0x11`.
- Call **F-I04.12** with \(G \ll 2\).

The second selector read must not be replaced by the first: intervening dependencies may change memory.

### 5. Additional byte clears and shifted arguments

After four calls, **F-I05.1–F-I05.4**:

1. Write zero to the byte at \(H + \texttt{0x0F}\).
2. Define **H1024** as \(H \times \texttt{0x400}\).
3. Read a byte at \(H1024 + 1\).
4. Write zero to the byte at that value plus `0x11`.
5. Call **F-I05.5** with four arguments: \(E \ll 23\), the same value again, the same value a third time, and \(E \ll 25\).
6. Call **F-I05.6**, forwarding its result to **F-I05.7**.
7. Execute four calls, **F-I05.8–F-I05.11**, producing **J**.
8. Read the byte at \(H1024 + 1\) again and retain it as **S**.
9. Write zero to the byte at \(S + \texttt{0x11}\).
10. Call **F-I05.12** with \(J \ll 2\).

S is an eight-bit selector, not a loaded pointer.

### 6. First word-clear group

After four calls, **F-I06.1–F-I06.4**:

- Write a zero word at \(S + \texttt{0x44}\).
- Write a zero word at absolute address `0x24`.
- Call **F-I06.5** with arguments \(0,\ \texttt{0x98},\ H1024,\ \texttt{0x98}\).
- Call **F-I06.6**, forwarding its result to **F-I06.7**.
- Call **F-I06.8**, forwarding its result to **F-I06.9**.
- Execute four calls, **F-I06.10–F-I06.13**, producing **K**.
- Write a zero word at \(4K + \texttt{0x44}\).
- Write a zero word at absolute address `0x24` again.
- Call **F-I06.14** with \(4K\).

The absolute-address writes are distinct events, not a single final-state assignment.

### 7. Split-result processing

A four-call run, **F-I07.1–F-I07.4**, yields a displayed 64-bit value. Let **L** be its low 32 bits and **U** its high 32 bits.

1. Call **F-I07.5** with \(L \ll 2\).
2. Write zero to the word at \(2U + \texttt{0x44}\).
3. Write zero to the word at absolute address `0x24`.
4. Call **F-I07.6** with zero, an auxiliary 32-bit result exposed by F-I07.5, and that auxiliary result shifted left by two.
5. Execute **F-I07.7** with no explicit arguments.
6. Call **F-I07.8** with \((H \mathbin{\&} \texttt{0x3FFFFF}) \ll 7\).

The upper half U comes from F-I07.4, whereas the auxiliary result comes from F-I07.5. They are not interchangeable.

### 8. Further word clearing and propagation

Define **M** as \(H \mathbin{\&} \texttt{0x3FFFFF}\).

A four-call run, **F-I08.1–F-I08.4**, produces **N**.

- Write zero to the word at \(4N + \texttt{0x44}\).
- Write zero to the word at absolute address `0x24`.
- Call **F-I08.5** with \(4N\).

A four-call run, **F-I08.6–F-I08.9**, produces **P**.

- Call **F-I08.10** with \(P \ll 2\), obtaining **Q**.
- Write zero to the word at \(4Q + \texttt{0x14}\).
- Call **F-I08.11** with Q three times, followed by \(M \ll 7\).

A four-call run, **F-I08.12–F-I08.15**, produces **R**.

- Write zero to the word at \(4R + \texttt{0x14}\).
- Call **F-I08.16** with \(4R\).

### 9. Scaled-word links

A four-call run, **F-I09.1–F-I09.4**, produces **T**.

- Retain **T4**, equal to \(4T\).
- Write T4 to the word at \(16T + \texttt{0x14}\).
- Call **F-I09.5** with T4.

A four-call run, **F-I09.6–F-I09.9**, produces **V**.

- Write retained T4 to the word at \(4V + \texttt{0x14}\).
- Call **F-I09.10** with \(4V\).

A four-call run, **F-I09.11–F-I09.14**, produces **W**.

- Write \(4W\) to the word at \(16W + \texttt{0x14}\).
- Call **F-I09.15** with \(4W\) and \(16W\).
- Call **F-I09.16**, forwarding its result as the first argument to **F-I09.17**; the second argument is \(M \ll 4\).

### 10. Masked-value word storage

A four-call run, **F-I10.1–F-I10.4**, produces **X**.

- Write \(M \times \texttt{0x80}\) to the word at \(4X + \texttt{0x14}\).
- Call **F-I10.5** with \(4X\).

Because M is at most `0x3FFFFF`, this stored value is at most `0x1FFFFF80`; this particular multiplication does not exceed 32 bits.

### 11. Final byte-selected updates

A four-call run, **F-I11.1–F-I11.4**, produces unsigned word **Y**.

Define **Z** as \(H \times \texttt{0x4000000}\), retaining 32 bits.

1. Read a byte at \(Z + 1\).
2. Write the low byte of \(Y \ll 2\) to the byte at that selector plus `0x11`.
3. Call **F-I11.5** with \(Y \ll 2\).
4. Call **F-I11.6**, forwarding its result to **F-I11.7**.
5. Execute four calls, **F-I11.8–F-I11.11**, producing **AA**.
6. Read the byte at \(Z + 1\) again.
7. Write the low byte of \((Y \mathbin{\&} \texttt{0x3FFFFFFF}) \gg 20\) to the newly selected destination.
8. Call **F-I11.12** with \(AA \ll 2\).
9. Execute four calls, **F-I11.13–F-I11.16**, producing unsigned word **AB**.
10. Read and retain byte **AC** from \(Z + 1\).
11. Write the low eight bits of \(4AB\) to the byte at \(AC + \texttt{0x11}\).
12. Call **F-I11.17** with AC, zero-extended to a word.
13. Call **F-I11.18**, forwarding its result to **F-I11.19**.
14. Execute four calls, **F-I11.20–F-I11.23**, producing **AD**.
15. Read a byte at \(8AB + 1\).
16. Write the low byte of \((AB \mathbin{\&} \texttt{0x3FFFFFFF}) \gg 20\) to that selector plus `0x11`.
17. Call **F-I11.24** with \(AD \ll 2\).
18. Execute four final calls, **F-I11.25–F-I11.28**.

The visible representation then terminates at its invalid-instruction boundary. No subsequent state, return, or cleanup is established.

## Side effects

### Direct memory effects

Visible accesses use field-like offsets `0x14`, `0x24`, `0x44`, and `0x74`, but these offsets do not establish a complete structure layout.

Byte-selected destinations have a particularly narrow range:

- A selector plus `0x11` addresses `0x11` through `0x110`.
- Retained selector S plus `0x44` addresses `0x44` through `0x143`.

These are literal calculated addresses in the representation. They must not silently be reinterpreted as offsets from an omitted object base.

The word at absolute address `0x24` is directly zeroed four times. The supplied warning indicates that this storage overlaps smaller symbolic storage; distinct-symbol independence must not be assumed.

### Ordering, aliasing, and interruption

The contract preserves separate reads and writes even when their formulas look similar. Calculated addresses may alias each other, absolute `0x24`, or memory touched by dependencies.

There is no visible synchronization, transaction boundary, rollback, or validation. A failure or nonreturning dependency can leave a partially modified state.

### Ownership

No allocation, transfer, retention, or release operation can be identified from the visible calls alone. Passing a value to a dependency does not establish ownership transfer. Likewise, no dependency may be assumed to preserve globals or refrain from freeing resources.

## Dependencies

All F-I identifiers denote unresolved indirect-call occurrences. Their implementations and semantic contracts are unavailable.

Established interactions include:

- Argument forwarding from explicit inputs and dependency results.
- Address-producing word results.
- A displayed 64-bit result split into two words.
- An auxiliary machine-register result used after a call.
- Calls interleaved with direct memory operations.

The initial target depends on incoming machine state. Subsequent target expressions are calculated from displayed execution-location values and fixed displacements. This does not establish stable public entry points, names, or equivalence among targets.

Unshown effects, exceptions, nonreturning behavior, memory ownership, global mutation, and machine-state preservation are all documented_only.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial shifted write.** If A is 3 and the incoming shift value is `0xFFFFFFFF`, the first word destination is `0x20`, and the written value is `0x1FFFFFFF`.

2. **Read-after-write distinction.** If B is 2, B4 is 8. The scaled-link write addresses `0x34`, while the following word read addresses `0x1C`. An implementation must not replace that read with the just-written value merely because both formulas contain `0x14`.

3. **Repeated selector reads.** If the first selector read at \(D16+1\) yields `0x20`, its selected write addresses `0x31`. If intervening dependencies change the selector to `0x30`, the later zero write addresses `0x41`, not `0x31`.

4. **Truncated high-field byte.** If D is `0x3FFFFFFF`, the shifted quantity before byte truncation is `0x3FF`; the byte written is `0xFF`. The retained H value is `0x1FFFFFFF`.

5. **Byte multiplication truncation.** If AB is `0x43`, the later byte value derived from \(4AB\) is `0x0C`, not a wider value of `0x10C`.

6. **Selector is not a pointer load.** If a selector byte is `0xFF`, its `0x11`-adjusted destination is address `0x110`; no additional pointer-sized dereference is implied.

7. **Partial progress.** If a dependency does not return after the first Input 3 write, the second Input 3 write is not reached. No rollback is established.

8. **Absolute-address write visibility.** If an observer can distinguish writes to address `0x24`, a full traversal exposes four direct zero-word writes there, subject to the supplied representation being executable as shown.

## Uncertainties

The following remain documented_only:

- Whether the supplied instruction decoding accurately describes executable control flow.
- The complete calling convention and origin of additional incoming state.
- Valid memory ranges, alignment requirements, byte order, and fault behavior.
- Whether low-address accesses are intentional, environment-mapped, or decoding artifacts.
- Concrete dependency identities and all unshown dependency effects.
- Exact executable semantics of signed-looking overflow and shifts.
- The true representation of the split result and auxiliary register result.
- Any behavior beyond the invalid-instruction boundary.
- Normal completion, cleanup, ownership rules, and error reporting.

The absence of visible branches does not establish unconditional completion. Every indirect call and memory operation is a potential boundary beyond which execution has not been established.

## Evidence

**Established static evidence:** The supplied representation shows the three explicit 32-bit inputs, additional incoming machine-state dependencies, ordered indirect calls, direct byte and word accesses, arithmetic masks and shifts, repeated absolute-address writes, split-result use, and an invalid-instruction ending.

**Inferred intent:** Repeated offsets and scaled values may represent linked records or encoded state. No such interpretation is required by this contract.

**Unknowns:** Dependency implementations, actual machine execution, complete control flow, runtime memory mapping, and ownership semantics are unavailable.

No runtime tests have been performed. The acceptance examples are arithmetic and ordering consequences of the supplied static representation, not observations of execution.
