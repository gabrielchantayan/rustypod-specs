<!-- Trusted provenance: F-6dfcdbe20955; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6dfcdbe20955

## Purpose

F-6dfcdbe20955 specifies an indefinitely repeated sequence of indirect dependency calls, computed-address memory accesses, and replacement of state carried into the next iteration.

The supplied static representation establishes these operations and their order. It does not establish an application-level purpose. In particular, the accessed storage is not established to be an object, device register, allocator structure, or protocol record.

A fully completed iteration performs **13 32-bit stores and 14 byte stores**, excluding dependency effects. There is no visible normal return, error return, termination condition, or recovery path. Continued repetition is conditional on every preceding call returning and every preceding access completing.

## Interface

### Explicit arguments

The represented interface contains four 32-bit argument positions:

| Position | Visible role |
|---|---|
| First | Passed to the first dependency call of each iteration; replaced near the end of the iteration. |
| Second | No explicit use in the visible body. |
| Third | No explicit use in the visible body. |
| Fourth | Used to form the first store address and an early dependency argument; replaced near the end of the iteration. |

Absence of explicit uses does not establish that the second and third positions are irrelevant to implicit calling-convention behavior.

### Effective state

The body additionally uses:

- A link/control value involved in selecting the first indirect target.
- A 32-bit **carried base**.
- A 32-bit **carried value**.
- An unresolved additional return-register value supplied to an early dependency.
- A 32-bit global value read late in each iteration.

The initial carried base, carried value, and control value are not initialized by the visible body. The origin and lifetime of the additional return-register value are unresolved; it is not established to be an independent entry argument.

The global is read at a particular point in each iteration, not captured at function entry.

### Results and prerequisites

There is no visible return-value contract. Observable results are dependency invocations, memory accesses, and loop-carried state changes.

Reaching any operation requires all preceding operations to complete. Successful continued execution additionally requires executable indirect targets and accessible storage for the computed accesses. No visible validation enforces those requirements.

### Widths and notation

A **word** is a 32-bit bit pattern; a **byte** is an 8-bit value. Dependency results explicitly used as pairs are 64 bits.

For the formulas below:

- \(L_n(x)\) means a left shift by \(n\), retaining the low 32 bits.
- Products and address additions use a 32-bit bit-pattern interpretation.
- Right shifts of unsigned words are logical.
- Byte selectors are zero-extended to values from 0 through 255.
- Byte stores retain only the low eight bits.
- A split 64-bit result supplies its low 32 bits first and high 32 bits second where a pair is passed.

This is a machine-oriented interpretation of the represented widths, not a claim that every signed-looking expression has defined high-level-language semantics.

A **selected-byte write through base \(B\)** comprises two ordered accesses: read one byte at \(B+1\), then write one byte at the zero-extended selector plus 17. The destination is not relative to \(B\).

## Behavior

The following phases describe one iteration. They are unconditional in the visible control flow, but each remains conditional on preceding calls and accesses completing.

Opaque dependency IDs identify argument-bearing call occurrences, not proven distinct functions. Additional calls are identified by their position in the sequence. “No explicit arguments” describes only the representation; it does not mean the call has no effective inputs.

### 1. Entry calls and initial publication

F-000000000001 receives the current first argument. Its target is derived from the incoming or loop-carried control value. Four no-explicit-argument dependency calls follow.

The body forms the **initial word** as \(L_{12}(\text{carried value})\), then stores it at:

\[
4096 \times \text{fourth argument}+20.
\]

F-000000000002 receives \(L_{14}(\text{fourth argument})\). One no-explicit-argument call follows.

F-000000000003 receives \(L_{28}(\text{initial word})\), followed by the unresolved additional return-register value. In the stated 32-bit model, its first argument is zero because the initial word already has 12 low zero bits.

Four no-explicit-argument calls follow. The final result selects a destination: the initial word is stored at four times that result plus 20. F-000000000004 then receives four times that result.

### 2. Initial selector operations and paired result

Four no-explicit-argument calls follow. Their final result is the **early index**; four times that index is retained as the **early scale**.

A selected-byte write through the carried base stores the low byte of:

\[
(\text{initial word}\mathbin{\&}\texttt{0x3fffffff})\gg20.
\]

F-000000000005 receives \(L_{24}(\text{carried value})\). One no-explicit-argument call produces a word. F-000000000006 receives that word and \(L_{28}(\text{early scale})\).

The next no-explicit-argument call produces a 64-bit result. F-000000000007 receives its low half and then its high half. Two no-explicit-argument calls follow, the second producing an index.

The body freshly reads the selector at carried base plus one and writes zero at selector plus 17. F-000000000008 receives \(L_2\) of the newly produced index.

### 3. Middle-word publication

Four no-explicit-argument calls follow. Their final result, shifted left two bits, becomes the **middle word**.

The middle word is stored at:

\[
4096 \times \text{early scale}+20.
\]

F-000000000009 receives, in order:

1. \(1024\times\text{early scale}\).
2. The same value.
3. \(L_{14}(\text{early index})\).
4. The middle word.

Four no-explicit-argument calls follow. Their final result selects another destination: four times that result plus 20 receives the middle word. F-00000000000a receives four times that result.

### 4. Two scaled-index publication pairs

Four no-explicit-argument calls produce the **first publication index**. Four times that index is stored at 16 times the index plus 20.

F-00000000000b receives the scaled value three times. Four no-explicit-argument calls produce a destination index. The same scaled value is stored at four times that destination index plus 20. F-00000000000c receives four times the destination index.

Four more no-explicit-argument calls produce the **second publication index**. Four times that index is stored at 16 times the index plus 20.

The body also retains the **quarter-space base**:

\[
\text{second publication index}\times\texttt{0x40000000}.
\]

F-00000000000d receives the second scaled value three times. Four no-explicit-argument calls produce the **second publication destination**. The second scaled value is stored at four times that destination plus 20. F-00000000000e receives four times that destination.

Under 32-bit arithmetic, the quarter-space base is one of `0x00000000`, `0x40000000`, `0x80000000`, or `0xc0000000`.

### 5. Repeated direct byte clearing

Four no-explicit-argument calls produce the **lookup token**.

The body directly stores zero at quarter-space base plus 17, then at quarter-space base plus one. Neither store uses a selector read.

F-00000000000f receives \(L_2(\text{lookup token})\). One no-explicit-argument call produces a word, which F-000000000010 receives.

Four no-explicit-argument calls produce an index. The two direct zero stores are repeated in the same order: offset 17, then offset one. F-000000000011 receives \(L_2\) of that index.

These are four separate byte stores. Intervening calls may change either location.

### 6. Lookup and zero-word publication

Four no-explicit-argument calls produce the **retained destination index**.

At 16 times that index plus 20, the body stores:

\[
L_{31}(\text{second publication destination}).
\]

This word is zero when that earlier destination is even and `0x80000000` when it is odd.

The body next reads a 32-bit **lookup word** from:

\[
((\text{lookup token}\mathbin{\&}\texttt{0x3fffffff})\gg20)+20.
\]

The starting address lies between 20 and 1043 inclusive; alignment is not established.

F-000000000012 receives four times the retained destination index. One no-explicit-argument call produces the **retained payload**. F-000000000013 receives:

\[
L_{28}(\text{lookup word}\gg22).
\]

Four no-explicit-argument calls produce an index. A zero word is stored at four times that index plus 20. F-000000000014 receives four times that index.

### 7. Selector-mediated payload publication

Four no-explicit-argument calls produce the **selector payload**.

A selected-byte write through the quarter-space base stores the low byte of \(L_2(\text{selector payload})\). F-000000000015 receives that full scaled word.

One no-explicit-argument call produces a word. F-000000000016 receives that word and \(L_{30}(\text{selector payload})\).

Four no-explicit-argument calls produce another index. The body reads the byte at quarter-space base plus one and retains it as the **captured selector**. At captured selector plus 17, it stores the low byte of:

\[
(\text{selector payload}\mathbin{\&}\texttt{0x3fffffff})\gg20.
\]

F-000000000017 receives \(L_2\) of the new index. Four no-explicit-argument calls follow.

The body then stores \(L_2(\text{retained payload})\) as a word at captured selector plus 20. This address uses the previously captured byte; there is no intervening selector reread for this word store.

### 8. Global snapshot and selector updates

The body reads the 32-bit global once and retains the **global snapshot**.

F-000000000018 receives four times the retained destination index. Four no-explicit-argument calls follow. The last returns a 64-bit result; its low half shifted left two bits becomes the **late scale**.

A fresh selected-byte write through the quarter-space base stores the low byte of the global snapshot shifted right 22 bits.

F-000000000019 receives the high half of the 64-bit result. One no-explicit-argument call produces a word. F-00000000001a receives that word and \(L_{28}(\text{late scale})\).

Four no-explicit-argument calls produce an index. A fresh selected-byte write through the quarter-space base writes zero. F-00000000001b receives \(L_2\) of that index.

The global is not reread for its byte publication, even if intervening dependencies modify it.

### 9. Late-word publication

Four no-explicit-argument calls produce an index. F-00000000001c receives \(L_2\) of that index twice.

Four no-explicit-argument calls produce a word. Its left shift by two becomes the **late word**, stored at:

\[
4096\times\text{late scale}+20.
\]

F-00000000001d receives \(L_{14}(\text{late scale})\). Four no-explicit-argument calls produce a destination index. The late word is stored at four times that destination index plus 20. F-00000000001e receives four times that index.

### 10. Final selector operations and next iteration

Four no-explicit-argument calls produce a 64-bit result. Its low half shifted left two bits becomes the **final scale**.

A selected-byte write through base \(16384\times\text{late scale}\) stores the low byte of:

\[
(\text{late word}\mathbin{\&}\texttt{0x3fffffff})\gg20.
\]

F-00000000001f receives the high half of the 64-bit result. One no-explicit-argument call produces a word, which F-000000000020 receives.

Another selector base is formed as:

\[
\text{final scale}\times\texttt{0x10000000}.
\]

Because final scale already contains a two-bit left shift, this base has the same four possible bit patterns listed for the quarter-space base.

One no-explicit-argument call produces a 64-bit result. F-000000000021 receives its low and high halves. Two no-explicit-argument calls follow, the second producing an index.

A selected-byte write through the newly formed base writes zero. F-000000000022 receives \(L_2\) of that index.

Four no-explicit-argument calls produce a result whose left shift by two becomes the **next carried value**. Four times final scale becomes the **next carried base**. A selected-byte write through that next carried base writes zero.

The fourth argument is replaced by the next carried value. F-000000000023 receives \(L_{10}(\text{final scale})\). One no-explicit-argument call produces a word. F-000000000024 receives that word and \(L_{28}(\text{retained payload})\).

Four final no-explicit-argument calls produce a result whose left shift by two becomes the next first argument. A fresh selected-byte write through the next carried base again writes zero.

Execution repeats. The next first-call target uses the control value explicitly established near the loop end, rather than a newly supplied entry value.

## Side effects

The visible per-iteration store inventory is:

| Phase | 32-bit stores | Byte stores |
|---|---:|---:|
| 1 | 2 | 0 |
| 2 | 0 | 2 |
| 3 | 2 | 0 |
| 4 | 4 | 0 |
| 5 | 0 | 4 |
| 6 | 2 | 0 |
| 7 | 1 | 2 |
| 8 | 0 | 2 |
| 9 | 2 | 0 |
| 10 | 0 | 4 |
| **Total** | **13** | **14** |

The body additionally performs ten selector-byte reads, one lookup-word read, and one global-word read. Counts exclude dependency effects, instruction fetching, and implicit execution-environment activity.

Access properties:

- Selected-byte destinations range from 17 through 272.
- The captured-selector word destination ranges from 20 through 275.
- Byte-derived word addresses need not be aligned.
- Writes and reads may overlap each other, selector storage, the global, or dependency-used memory.
- Distinct operations remain distinct even when their addresses coincide.
- No bounds checks, rollback, synchronization, or atomicity guarantee is visible.

No ownership transfer, allocation, or release is established by the visible body. Dependency arguments are words; their interpretation as integers, addresses, handles, or ownership-bearing values is unresolved.

## Dependencies

F-000000000001 through F-000000000024 identify the argument-bearing occurrences described above. Every intervening no-explicit-argument call is also an unresolved indirect dependency, distinguished by its ordered position.

These identifiers do not assert target uniqueness or semantic equivalence. Target resolution and complete calling contracts are **documented_only**.

Dependencies are not established to preserve globals, selector storage, unspecified registers, or other memory. They are also not established to return normally, avoid callbacks, avoid blocking, or refrain from allocating or releasing resources.

The first target depends on control state. Later targets are predominantly formed from fixed control-location-derived expressions. Executable validity is unresolved.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial publication.** With fourth argument 2 and carried value 3, the initial word is `0x00003000`, stored at `0x00002014`. F-000000000002 receives `0x00008000`, provided preceding operations complete.

2. **Width-sensitive argument.** Any initial word formed by a 12-bit left shift produces zero when subsequently shifted left 28 bits in the 32-bit model. F-000000000003 still receives its unresolved second argument.

3. **Selector addressing.** A selector byte of `0xff` directs a selected-byte write to address 272, regardless of the selector-storage base. It does not supply a full pointer.

4. **Fresh versus captured selectors.** Separate selected-byte writes that read 4 and 9 target addresses 21 and 26. In phase 7, a captured selector of 7 directs the later word store to address 27 even if intervening calls change selector storage.

5. **Byte truncation.** A selector payload of `0x00000041` yields scaled word `0x00000104`; phase 7’s first byte publication stores `0x04`.

6. **Earlier-destination parity.** A second publication destination of 5 makes phase 6’s parity-dependent word `0x80000000`; a destination of 6 makes it zero.

7. **64-bit pair order.** A paired result of `0x1122334455667788` is passed as `0x55667788`, then `0x11223344`.

8. **Global snapshot.** If the captured global is `0x00400000`, its later byte publication is `0x01`, even if a dependency changes the global before that publication.

9. **Store inventory.** A complete iteration has 13 visible word stores and 14 visible byte stores. Aliasing does not reduce those operation counts.

10. **Failure and continuation.** An inaccessible address has no specified error-code response. If all operations complete instead, the final clear is followed by another iteration, not a normal return.

## Uncertainties

The following remain **documented_only**:

- Application purpose and data-structure interpretation.
- Complete calling convention and implicit inputs.
- Initial carried state and unresolved return-register provenance.
- Dependency identities, preservation rules, and exceptional behavior.
- Address mapping, endianness, alignment tolerance, and fault behavior.
- Ordinary-memory versus device-memory interpretation.
- Global boundaries, aliasing, concurrency, and reentrancy.
- Exact instruction-level arithmetic semantics beyond the represented widths.
- Completeness of the recovered control flow.

The supplied representation reports an unreachable region being removed and overlapping global symbols. These are limits on completeness and object-boundary confidence, not evidence of a particular alternative execution.

## Evidence

Established static evidence consists of the supplied representation’s explicit widths, ordered indirect calls, arithmetic expressions, memory accesses, state replacements, and unconditional repetition. The store totals follow from the phase inventory above.

Descriptive labels identify observed value roles, not source-level objects. Phase groupings aid explanation; they do not establish transactions or application intent.

No runtime tests, dependency resolution, executable inspection, or independent implementation verification have been performed.
