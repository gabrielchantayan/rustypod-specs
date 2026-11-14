<!-- Trusted provenance: F-4ab14a19afe5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4ab14a19afe5

## Purpose

F-4ab14a19afe5 describes an ordered sequence of indirect calls, computed-address reads, and memory updates. Its visible effects include publishing scaled call results, storing the third explicit input unchanged at two computed word locations, updating byte-selected destinations, clearing computed word locations, and making four separate zero assignments to the global associated with address `0x24`.

The access width of those four global assignments is **unknown**. They are distinct from the explicitly typed 32-bit memory accesses elsewhere, including computed-address writes whose address expression ends in `+ 0x24`.

No higher-level purpose is established. Initialization, registration, encoding, and state clearing are possible interpretations, not contractual guarantees.

The supplied static representation reports invalid instruction data and terminates at a truncated control-flow boundary. This document specifies the represented prefix, conditional on execution reaching each operation. It does not establish successful return, transaction completion, or an intended continuation.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits, uninterpreted | Forwarded to the first indirect call. |
| Second input | 32 bits, uninterpreted | Forwarded to the first indirect call. |
| Third input, designated **P** | Unsigned 32 bits | Forwarded to the first call, logically shifted right by three, and stored unchanged at two computed word locations. |

The visible body does not explicitly dereference the first two inputs. Their interpretation by the first dependency is unknown.

### Additional entry-state dependencies

Behavior also depends on:

- An unsigned 32-bit entry-state value whose logical right shift by three becomes the first call’s fourth argument.
- An uninterpreted 32-bit entry-state value used as the payload of the first direct memory write.
- An entry link/control value used to derive the first indirect target.
- Memory contents at computed addresses, including unsigned byte selectors.
- Indirect-call results, including one represented 64-bit result and a later secondary-register value.

These dependencies do not establish additional portable parameters or a complete calling convention.

### Output and termination

The represented interface has no ordinary return value. No normal return is visible. The endpoint is an invalid-instruction termination marker after the final indirect call sequence.

Whether the underlying execution traps there, reaches another continuation, or can reach this prefix at all remains `documented_only`.

### Width and arithmetic conventions

A **word** below means an explicitly represented 32-bit value or memory access. A **byte** means an unsigned 8-bit selector read or an 8-bit memory write. These definitions do not assign a width to the global associated with `0x24`.

The following mathematical labels identify retained values:

| Label | Meaning |
|---|---|
| S, T, V | Scaled early call results, each formed by multiplication by four |
| N | Result retained immediately before the first direct store of P |
| H | `(N & 0x3fffffff) >> 1` |
| B | `16 × N` |
| R | A later result retained across several call sequences |
| K | `1024 × H` |
| M | `H & 0x3fffff` |
| J | `0x4000000 × H` |

Address formulas describe the represented computations, not unbounded integer arithmetic or guaranteed valid addresses. Unsigned 32-bit operations have their represented 32-bit range; signed multiplication, signed left shifts, signed conversions, and address interpretation require care. Mathematical examples below avoid overflow unless stated otherwise.

Logical right shifts of unsigned values, masks, and byte truncation are established. Pointer width, alignment rules, and byte order are not supplied.

## Behavior

### Dependency notation and ordering

Opaque identifiers `D-001` through `D-031` and `E-001` through `E-007` identify ordered dependency sequences within this contract. They do not assert distinct callee identities.

Unless expressly described otherwise, a sequence contains **five indirect calls**:

- Its first call receives the arguments listed.
- Four further calls show no explicit arguments.
- A result attributed to the sequence belongs to its fifth call.

“No explicit arguments” does not mean that a callee has no inputs: registers, memory, globals, and other ambient state may matter.

The sequences occur in numeric order, with the `E-` sequences following `D-031`. Described reads and writes occur between the stated sequences. Calls may observe or change memory; repeated reads are therefore independent observations.

### Initial publications

| Sequence | Arguments to its first call | Direct effect following the sequence |
|---|---|---|
| `D-001` | First input, second input, P, entry-state value logically shifted right by three | For result A, write the retained entry-state payload as a word at `4 × A + 0x14`. |
| `D-002` | `4 × A` | For result C, retain S as `4 × C` and write S as a word at `16 × C + 0x14`. |
| `D-003` | S | For result E, write S as a word at `4 × E + 0x14`. |
| `D-004` | `4 × E` | For result G, retain T as `4 × G` and write T as a word at `16 × G + 0x14`. |

The value `P >> 3` is retained before `D-003`; it is not obtained by a later memory read.

`D-005` contains only **two calls**. Its first receives T; its second produces the word supplied to `D-006`.

| Sequence | Arguments to its first call | Direct effect following the sequence |
|---|---|---|
| `D-006` | Second result of `D-005` | For result I, write `(T & 0x3fffffff) >> 1` as a word at `4 × I + 0x14`. |
| `D-007` | `4 × I` | For result L, retain V as `4 × L` and write V as a word at `16 × L + 0x14`. |
| `D-008` | A fresh word read at `V + 0x14`, logically shifted right by three | For result Q, write V as a word at `4 × Q + 0x14`. |
| `D-009` | `4 × Q` | Retain result N; write P as a word at `0x80000 × (P >> 3) + 0x24`. |

The read supplying `D-008` is at `4 × L + 0x14`, whereas the preceding write is at `16 × L + 0x14`. They are not interchangeable observations.

### Third-input publication and early byte updates

`D-010` receives B and `N << 2`. After its five calls, P is written as a word at `B + 0x24`. A word is then read from `B + 0x74` and passed to `D-011`.

`D-011` returns the retained result R. Afterward, an unsigned byte is read from `B + 1`. That byte selects an absolute destination: the low eight bits of `(N & 0x3fffffff) >> 20` are written at the selector plus `0x11`.

The destination is between `0x11` and `0x110`, inclusive. The selector is neither a pointer-sized load nor an offset added back to B.

`D-012` contains **two calls**. Its first receives `R << 2`; its second result supplies `D-013`. After `D-013`, a fresh byte read from `B + 1` selects a byte destination, again by adding `0x11`, and zero is written there.

`D-014` receives the result of `D-013` shifted left by two. Afterward:

- A zero byte is written at `H + 0x0f`.
- A byte is read from `K + 1`.
- A zero byte is written at that selector plus `0x11`.

`D-015` contains **two calls**. Its first receives three copies of `R << 23`, followed by `R << 25`. Its second result supplies `D-016`.

After `D-016`, a fresh byte from `K + 1` is retained as Z, and zero is written as a byte at `Z + 0x11`.

`D-017` receives the result of `D-016` shifted left by two. Afterward, a zero word is written at `Z + 0x44`, followed by the first zero assignment to the global associated with `0x24`.

The word destination uses retained Z, not a new read of `K + 1`. The following global assignment has unknown access width.

### Repeated clearing and global assignments

`D-018` contains **four calls**:

| Call position | Explicit arguments or result use |
|---|---|
| First | Receives zero, `0x98`, K, and `0x98`. |
| Second | No explicit arguments; produces a word. |
| Third | Receives that word. |
| Fourth | No explicit arguments; produces the word supplied to `D-019`. |

For result U from `D-019`, a zero word is written at `4 × U + 0x44`. This is followed by the second zero assignment to the global associated with `0x24`.

`D-020` receives `4 × U` and returns a represented 64-bit quantity. Its low and high 32-bit halves have separate uses:

- The low half, interpreted through a represented signed 32-bit conversion and shifted left by two, supplies `D-021`.
- `D-021` is a **single call**.
- After that call, a zero word is written at twice the signed interpretation of the retained high half, plus `0x44`.
- The third zero assignment to the global associated with `0x24` follows.

The high half belongs to `D-020`; it is not the result of `D-021`. Signed conversion and shift behavior at problematic bit patterns remain unresolved.

`D-022` contains **two calls**. The first receives zero, a represented secondary-register value associated with the intervening call, and that value shifted left by two. The second shows no explicit arguments. The exact calling-convention meaning and lifetime of the secondary value are unknown.

`D-023` receives `M << 7`. For result W, a zero word is written at `4 × W + 0x44`, followed by the fourth zero assignment to the global associated with `0x24`.

Thus, four distinct global zero assignments are represented, each after a specific word clear and separated from the others by dependency calls. Their access width is not established. They must not be collapsed into a single event or described as four proven 32-bit stores.

### Final word publications

`D-024` receives `4 × W`. Its result shifted left by two supplies `D-025`, which is a **single call** returning Y. A zero word is then written at `4 × Y + 0x14`.

| Sequence | Arguments to its first call | Direct effect following the sequence |
|---|---|---|
| `D-026` | Y, Y, Y, `M << 7` | For result AA, write a zero word at `4 × AA + 0x14`. |
| `D-027` | `4 × AA` | For result AC, retain AD as `4 × AC`; write AD as a word at `16 × AC + 0x14`. |
| `D-028` | AD | For result AE, write AD as a word at `4 × AE + 0x14`. |
| `D-029` | `4 × AE` | For result AG, write `4 × AG` as a word at `16 × AG + 0x14`. |

`D-030` contains **two calls**. Its first receives `4 × AG` and `16 × AG`; its second produces a word.

`D-031` receives that word and `M << 4`. For result AI, it is followed by a word write of `128 × M` at `4 × AI + 0x14`.

M is at most `0x3fffff`. Consequently, `M << 7` and `128 × M` are equal and fit within 29 unsigned bits. This equality does not depend on overflow.

### Final byte updates

| Sequence | Arguments to its first call | Subsequent direct effects |
|---|---|---|
| `E-001` | `4 × AI` | Retain result AJ. Read a byte at `J + 1`; write the low eight bits of `AJ << 2` at the selector plus `0x11`. |
| `E-002` | `AJ << 2` | Two calls only; its second result supplies `E-003`. |
| `E-003` | Second result of `E-002` | Read `J + 1` again; write the low eight bits of `(AJ & 0x3fffffff) >> 20` at the new selector plus `0x11`. |
| `E-004` | Result of `E-003` shifted left by two | Retain result AK. Read and retain byte AL at `J + 1`; write the low eight bits of `4 × AK` at `AL + 0x11`. |
| `E-005` | AL, zero-extended to 32 bits | Two calls only; its second result supplies `E-006`. |
| `E-006` | Second result of `E-005` | Read a byte at `8 × AK + 1`; write the low eight bits of `(AK & 0x3fffffff) >> 20` at the selector plus `0x11`. |
| `E-007` | Result of `E-006` shifted left by two | Five calls; no further direct memory operation is shown before the invalid-instruction endpoint. |

The payload after `E-003` still derives from AJ. The payload and selector-source address after `E-006` derive from retained AK, not from the result of `E-006`.

## Side effects

### Access forms

| Represented access | Established width |
|---|---:|
| Explicit computed-address accesses ending in `+ 0x14`, `+ 0x24`, `+ 0x44`, or `+ 0x74` | 32 bits |
| Selector reads at a computed base plus one | 8 bits, unsigned |
| Writes at a byte selector plus `0x11` | 8 bits |
| Write at `H + 0x0f` | 8 bits |
| Four zero assignments to the global associated with `0x24` | **Unknown** |

Offsets do not establish a shared structure, allocation, or ownership boundary. In particular, explicit word writes at computed addresses ending in `+ 0x24` do not determine the width of the separately represented global assignments, even if an address happens to coincide.

Some word destinations can be unaligned, notably `Z + 0x44`. Alignment requirements and unaligned-access behavior are unknown.

### Aliasing and visibility

No non-aliasing guarantee is established. Direct writes may affect later selector reads, dependency state, other destinations, or control data where the environment permits such mappings.

The supplied warning about overlapping global symbols prevents treating the global associated with `0x24` as a proven independent object. Without its declaration, the affected byte range of each global assignment is unknown.

The specified order is the represented operation order. Atomicity, barriers, hardware visibility, volatile semantics, and concurrent-access guarantees are not established.

### Ownership and failures

No direct allocation, release, or ownership-transfer operation is identified. Dependencies may allocate, release, retain, or mutate resources. Neither preservation nor ownership behavior may be assumed.

No validation, bounds check, status check, rollback, or recoverable error branch is visible. If an access or dependency does not complete, later effects are not established. Earlier effects have no visible rollback.

Executable indirect targets and accessible computed memory are prerequisites for completing the prefix, not guarantees made by this function.

## Dependencies

All visible calls are indirect. The opaque dependency identifiers above denote ordered sites or sequences, not resolved function identities.

Static evidence establishes that:

- The first target depends on entry control state.
- Later targets use embedded control-location values and fixed displacements.
- The displacements are not uniform.
- Some results feed later arguments or address calculations.
- Calls with unused results remain observable dependencies.

No dependency implementation is supplied. Its argument interpretation, ambient-state inputs, global preservation, ownership behavior, failure behavior, and ability to return are `documented_only`.

The representation retains particular values across calls. That observation does not establish that dependencies preserve arbitrary globals or memory.

## Acceptance examples

These are conditional static review examples, not executed tests. Each assumes the relevant prefix is reached, dependencies complete, and required accesses are permitted.

1. **Initial forwarding.** With entry-state value `0x28`, the first call’s fourth argument is `5`. The first three arguments remain the explicit inputs.

2. **Unchanged third-input payload.** With P equal to `0x18`, the first P publication writes the full 32-bit value `0x18` at `0x180024`, not the shifted value `3`.

3. **Different address scales.** If `D-007` returns `2`, it writes word value `8` at `0x34`. The argument to `D-008` comes from a separate word read at `0x1c`, shifted right by three.

4. **Absolute byte selection.** A selector value of `0xff` chooses byte destination `0x110`. The destination is not relative to B, K, or J.

5. **Fresh reads across calls.** If the relevant reads at `B + 1` return `3` and then `7`, the writes target `0x14` and `0x18`, respectively. Reusing the first selector would change the specified behavior.

6. **Byte truncation.** If `(AJ & 0x3fffffff) >> 20` equals `0x3ff`, the corresponding byte payload is `0xff`.

7. **Retained selector.** If Z is `5`, the word clear after `D-017` targets `0x49`, even if dependencies have since changed `K + 1`.

8. **Global assignment width remains unresolved.** After each of the four designated word clears, the corresponding zero assignment to the global associated with `0x24` must remain an ordered event. The evidence does not justify requiring a one-, two-, four-, or eight-byte access for that event.

9. **No final-zero guarantee.** Completion of the fourth global assignment does not prove the global remains zero at the endpoint. Later dependencies may modify it.

10. **Bounded masked scaling.** If M is `0x3fffff`, the final word payload is `0x1fffff80`. This particular scaling fits within 32 bits.

## Uncertainties

The following remain `documented_only`:

- Validity and reachability of the represented instruction stream.
- Completeness of the signature and calling convention.
- Exact signed-overflow, signed-shift, and signed-conversion behavior.
- Pointer width, alignment requirements, and byte order.
- The declaration, type, access width, and storage extent of the global associated with `0x24`.
- The meaning of fixed offsets and computed address formulas.
- Actual dependency identities and target aliasing.
- Ambient inputs to calls without explicit arguments.
- Dependency effects on globals, memory, ownership, and resource lifetime.
- Exceptions, traps, nonlocal exits, and non-returning calls.
- Actual terminal behavior beyond the truncated boundary.

An initialization or clearing interpretation is inferred intent only. It does not resolve any of these uncertainties.

## Evidence

Established static evidence consists of the supplied representation’s declared input widths, explicit memory-access types, call ordering, masks, shifts, result retention, address calculations, and termination warning.

The four assignments to the global associated with `0x24` have no supplied declaration establishing their type or access width. Explicit 32-bit pointer accesses elsewhere do not supply that missing evidence. The overlapping-symbol warning also does not establish width; it adds uncertainty about storage boundaries and aliasing.

No implementation access, independent instruction inspection, dependency inspection, runtime execution, or tests have been performed. Acceptance examples are conditional arithmetic and ordering consequences, not observed execution results.
