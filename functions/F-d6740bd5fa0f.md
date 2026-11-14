<!-- Trusted provenance: F-d6740bd5fa0f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d6740bd5fa0f

## Purpose

This function is specified as an ordered sequence of indirect calls, memory reads, and memory writes. The supplied static representation does not establish an application-level purpose such as allocation, scheduling, initialization, or serialization.

The contract is conditional: each operation is reached only if preceding calls return and preceding memory accesses complete. No explicit branches, loops, validation, error-handling paths, or rollback operations appear.

Normal completion is not established. The final call targets address zero; the represented return occurs only if that call returns.

## Interface

### Explicit arguments and result

The represented interface has four 32-bit arguments and no explicit return value.

| Argument | Established direct use |
|---|---|
| 1 | None shown. |
| 2 | Supplied to F-000000000001. |
| 3 | None shown. |
| 4 | Interpreted as unsigned, logically shifted right by 23, and supplied to F-000000000003. |

The absence of a direct use does not establish machine-level irrelevance. The actual calling convention and hidden dependency inputs are unresolved.

### Implicit inputs and state

Execution additionally uses:

- An incoming link-related value when forming the first call target.
- An implicit 32-bit value supplying an early byte-write payload.
- An implicit address supplying the first byte lookup.
- A second implicit 32-bit value supplying two later lookup addresses.
- A secondary-register value captured after F-000000000016.
- A secondary-register pointer used near the end.
- Three global state views:
  - **G-control:** receives two represented 32-bit writes.
  - **G-anchor:** supplies a saved address.
  - **G-upper24:** supplies the upper 24 bits of the final G-control value.

These are descriptive labels, not established public parameters or independent storage objects. Global storage may overlap. Initialization, accessibility, and provenance are `documented_only`.

### Widths and notation

The following notation describes values, not implementation requirements:

- **scale4(x):** low 32 bits of four times x.
- **scale16(x):** low 32 bits of sixteen times x.
- **low8(x):** least significant eight bits.
- **low32(pair)** and **high32(pair):** the two 32-bit halves of a represented 64-bit result.
- **x << n:** a left shift, retaining the low 32 bits under the conditional machine-arithmetic model.
- **x >> n:** an unsigned logical right shift wherever used below.

Byte accesses are 8 bits; word accesses are 32 bits; the final halfword access is 16 bits. Byte lookups are unsigned and zero-extended before address formation.

Some scaling and left shifts have signed intermediates in the supplied representation. Their exact overflow behavior is not established. The modular definitions above provide a conditional 32-bit machine interpretation, not a claim that all corresponding signed-language expressions have defined semantics. Address additions likewise require confirmation of the underlying machine semantics.

No endianness, alignment policy, atomicity, or memory-mapped-I/O behavior is established.

## Behavior

### Trace conventions

F-000000000001 through F-000000000114 are opaque dependency IDs assigned to **call occurrences**. They do not establish 114 distinct implementations.

A range of calls executes in ascending ID order. “No explicit arguments” means none are shown; it does not exclude hidden register or stack inputs. Only results subsequently consumed by this function are identified.

Saved-value labels below belong to this contract. They are not source identifiers.

### 1. Entry calls and first byte-selected write

1. F-000000000001 receives argument 2. Its target is the incoming link-related value plus `0xFEC`.
2. F-000000000002 has no explicit arguments.
3. F-000000000003 receives argument 4 >> 23.
4. F-000000000004 through F-000000000007 have no explicit arguments. Save the last 32-bit result as **A**.
5. Read a byte at the implicit address plus 1. Add `0x11` to its zero-extended value to obtain a destination. Write low8 of the first implicit 32-bit value >> 22 to that destination.
6. F-000000000008 receives scale4(A).

The destination in step 5 comes from the byte itself, not from a pointer-sized value stored at the lookup address. Its numeric range is `0x11` through `0x110`.

### 2. Duplicated argument and secondary-register capture

1. F-000000000009 through F-000000000012 have no explicit arguments. Save the last result as **B**.
2. F-000000000013 receives scale4(B), scale4(B).
3. F-000000000014 produces a 32-bit value **C**.
4. F-000000000015 receives C.
5. F-000000000016 has no explicit arguments.
6. Read a byte at `(second implicit value >> 9) + 1`. Write a zero byte at the zero-extended lookup result plus `0x11`.
7. Capture the represented secondary-register value as **S**.
8. F-000000000017 receives S.

The representation does not establish a conventional function-result signature for S. The lookup address in step 6 can range from 1 through `0x800000` under the unsigned 32-bit interpretation.

### 3. Repeated lookup and first G-control update

1. F-000000000018 produces a 32-bit value **D**.
2. F-000000000019 receives D, C >> 23.
3. F-000000000020 through F-000000000023 have no explicit arguments. Save the last result as **E**.
4. Read the byte at `(second implicit value >> 9) + 1` again. Save its zero-extended value as **K**, then write a zero byte at K plus `0x11`.
5. F-000000000024 receives scale4(E).
6. F-000000000025 through F-000000000028 have no explicit arguments. Save the last result as **F**.
7. F-000000000029 receives scale4(F), scale4(F).
8. F-000000000030 produces a 32-bit value **G**.
9. Write G to G-control.
10. F-000000000031 receives G.
11. F-000000000032 has no explicit arguments.
12. Read a word at K plus `0x74`; write its low byte at K plus 1.

The two lookups using the second implicit value are separate memory reads, separated by calls. They cannot be replaced by a cached lookup result. Conversely, the later accesses through K use the saved second result rather than another lookup.

### 4. First pair result and repeated byte refresh

1. F-000000000033 receives S << 11.
2. F-000000000034 produces a represented 64-bit pair.
3. F-000000000035 receives, in order, its low half, its high half, and G >> 23.
4. F-000000000036 through F-000000000039 have no explicit arguments. Save **J = scale4(last result)**.
5. Read a word at J plus `0x74`; write its low byte at J plus 1.
6. F-000000000040 receives J.
7. F-000000000041 through F-000000000044 have no explicit arguments. Save the last result as **L**, and save **U = scale4(L)**.
8. Read a word at J plus `0x74` again; write its low byte at J plus 1 again.
9. F-000000000045 receives S << 12, U, U, L << 18.

The second word read uses the saved J but fresh memory contents. Intervening calls may change the value observed. Nothing establishes that the two byte writes agree.

### 5. Captured word and two destination scales

1. F-000000000046 produces a word passed to F-000000000047.
2. F-000000000048 produces a word passed to F-000000000049.
3. F-000000000050 through F-000000000053 have no explicit arguments. Save **N = scale4(last result)**.
4. Read a word at N plus `0x74`, saving the full value as **V**. Write low8(V) at N plus 1.
5. F-000000000054 receives N.
6. F-000000000055 through F-000000000058 have no explicit arguments.
7. Write V at `scale16(last result) + 0x14`.
8. F-000000000059 receives `(U >> 2) << 16`.
9. F-000000000060 produces a word passed to F-000000000061.
10. F-000000000062 produces a word passed to F-000000000063.
11. F-000000000064 through F-000000000067 have no explicit arguments. Save **P = scale4(last result)**.
12. Write V at P plus `0x14`.
13. F-000000000068 receives P.

Both full-word writes use saved V. Neither rereads N plus `0x74`. The first destination uses a sixteen-byte scale; the second uses a four-byte scale.

The transformation passed to F-000000000059 operates on U after its earlier scaling and truncation. It must not be replaced with a transformation of untruncated L without proving equivalence.

### 6. Scaled-value write and zero-word write

1. F-000000000069 through F-000000000072 have no explicit arguments. Save **Q = scale4(last result)**.
2. Write Q at `(U & 0xFFFFFFFC) + 0x14`.
3. F-000000000073 receives Q, Q.
4. F-000000000074 produces a word passed to F-000000000075.
5. F-000000000076 produces a word passed to F-000000000077.
6. F-000000000078 through F-000000000081 have no explicit arguments. Save **R = scale4(last result)**.
7. Write a zero word at R plus `0x14`.
8. F-000000000082 receives R.

Under the modular model, scaling already clears U’s lowest two bits. The explicit mask remains part of the represented address calculation.

### 7. Pair-derived writes and captured global address

1. F-000000000083 through F-000000000086 have no explicit arguments. The last produces a represented 64-bit pair.
2. Save **T = scale4(low32(pair))** and **H = high32(pair)**.
3. Write T at `scale4(H) + 0x14`.
4. Read G-anchor and save its value as **Anchor**.
5. F-000000000087 receives T, H, T, T.
6. F-000000000088 through F-000000000091 have no explicit arguments. Save **X = scale4(last result)**.
7. Write T at X plus `0x14`.
8. Read a word at Anchor plus `0x14`, saving it as **W**.
9. F-000000000092 receives X.

G-anchor is read after the first T write and before F-000000000087. Later changes to G-anchor do not themselves replace saved Anchor. The memory reached through Anchor is read only after F-000000000091 and the second T write.

Aliasing may affect both the captured global value and W; no disjointness is established.

### 8. Two further byte-selected writes

1. F-000000000093 through F-000000000096 have no explicit arguments. The last produces a represented 64-bit pair. Save its low half as **Y** and high half as **Z**.
2. Read a byte at `(V >> 23) + 1`. Write low8(W) at the zero-extended lookup result plus `0x11`.
3. F-000000000097 receives Z, Z, Z, scale4(Y).
4. F-000000000098 produces a word passed to F-000000000099.
5. F-000000000100 through F-000000000103 have no explicit arguments. Save the last result as **AA**.
6. Read a byte at `((Y & 0x3FFFFFFF) >> 21) + 1`. Write low8(W >> 22) at the zero-extended lookup result plus `0x11`.
7. F-000000000104 receives scale4(AA).

Each lookup address ranges from 1 through 512. Each resulting destination ranges from `0x11` through `0x110`. Both writes use saved W, not a fresh read through Anchor.

### 9. Final memory updates and zero-target call

1. F-000000000105 through F-000000000108 have no explicit arguments. Save the last result as **AB**.
2. F-000000000109 receives scale4(AB), scale4(AB).
3. F-000000000110 has no explicit arguments.
4. F-000000000111 produces a word passed to F-000000000112.
5. F-000000000113 produces an 8-bit value **FinalByte**.
6. Write a zero byte through the represented secondary-register pointer.
7. Read G-upper24. Write G-control with FinalByte as its least significant byte and that read value as its upper 24 bits.
8. Write a zero halfword at `(Y & 0x3FFFFFFF) >> 7`.
9. F-000000000114 calls address zero with three explicit zero arguments.
10. Return only if that call returns normally.

The pointer clear precedes the G-upper24 read. If those storage locations overlap, the clear may affect the composed value.

The halfword destination ranges from zero through `0x7FFFFF`; its calculation does not guarantee alignment.

## Side effects

The function contains 114 represented calls. Their effects are additional to, and not bounded by, the explicit accesses described here.

There are **six explicit 32-bit writes to calculated addresses, excluding the two G-control writes**:

| Order | Value written | Destination |
|---|---|---|
| 1 | V | scale16(result of F-000000000058) plus `0x14` |
| 2 | V | P plus `0x14` |
| 3 | Q | `(U & 0xFFFFFFFC) + 0x14` |
| 4 | Zero word | R plus `0x14` |
| 5 | T | scale4(H) plus `0x14` |
| 6 | T | X plus `0x14` |

Other explicit effects include byte lookups, word-to-byte copies, byte clears, two G-control writes, one G-anchor read, one later word read through saved Anchor, one G-upper24 read, and the final halfword clear. The secondary-register pointer clear is byte-sized and is not an additional word write.

No allocation, release, ownership transfer, synchronization, or cleanup operation is explicitly established. Callees may perform such operations; their absence from the visible trace does not establish their absence from execution.

There is no transactional guarantee. A fault, nonreturning dependency, or control transfer can leave earlier effects visible. No rollback or explicit error result is represented.

## Dependencies

F-000000000001 through F-000000000113 are unresolved indirect-call occurrences. Their explicit argument order and consumed results are specified above. Most targets are formed from represented location-derived values and fixed offsets. The first target additionally depends on incoming execution context.

F-000000000114 explicitly targets address zero.

For every occurrence, the following remain unknown:

- Callable identity, instruction mode, and complete calling convention.
- Hidden inputs and register-preservation rules.
- Effects on globals, other memory, and resource lifetimes.
- Whether memory is acquired, retained, transferred, invalidated, or released.
- Whether control returns normally, faults, terminates, or transfers elsewhere.

A represented 64-bit result establishes two consumed 32-bit halves, not a verified scalar-return ABI. No dependency may be assumed to preserve globals or memory merely because its effects are unspecified.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes execution reaches the relevant operations and the required accesses complete.

1. **Unsigned argument extraction.** With argument 4 equal to `0xFF800000`, F-000000000003 receives `0x1FF`, not a sign-extended value or a single extracted bit.

2. **Byte-selected destination and truncation.** If the initial lookup returns `0x20` and the payload-producing implicit value is `0xFFC00000`, the write targets `0x31` and stores `0xFF`.

3. **Independent lookup reads.** If the first lookup using the second implicit value returns `0x10` and the later lookup returns `0x30`, the zero-byte destinations are `0x21` and `0x41`. Saved K is `0x30`.

4. **Fresh word reads.** If J plus `0x74` initially contains `0x12345678`, the first refresh stores `0x78`. If an intervening callee changes that word to `0xABCDEF01`, the second refresh stores `0x01`.

5. **Saved full-word value.** If V is `0xA1B2C3D4`, the immediate byte copy stores `0xD4`, and both subsequent V word writes store `0xA1B2C3D4`. Changing the original source memory does not itself change saved V.

6. **Different destination scales.** A result of 3 from F-000000000058 gives destination `0x44`. A result of 3 from F-000000000067 gives destination `0x20`.

7. **Anchor aliasing.** If Anchor plus `0x14` equals X plus `0x14`, W observes the preceding T write, assuming ordinary coherent memory and no intervening external modification.

8. **Final composition and ordering.** If the G-upper24 read supplies `0x123456` and FinalByte is `0xAB`, G-control receives `0x123456AB`. The relevant upper value is the one read after the secondary-pointer byte clear.

9. **Unaligned or zero halfword address.** Y equal to zero produces halfword destination zero. Y equal to `0x80` produces destination 1. Accessibility and alignment handling remain unresolved; neither case is validated before the write.

10. **Terminal-call failure.** If calling address zero faults, normal return is not required. Earlier writes are not undone by any represented operation.

## Uncertainties

### Established static evidence

The supplied representation establishes the explicit operation order, operand transformations, access widths, consumed results, six calculated-address word writes, two G-control writes, and absence of explicit conditional control flow.

### Inferred intent

Repeated copies from offset `0x74` to offset 1 may refresh byte-sized state. Writes at offset `0x14` may update related records. These are interpretations only; they establish neither object layouts nor application semantics.

### `documented_only`

Unresolved matters include:

- Authenticity of the represented function boundary and callable entry conditions.
- Provenance and validity of implicit register inputs and secondary-register values.
- Exact instruction-level overflow, signed-shift, and address-formation semantics.
- Memory accessibility, alignment, volatility, endianness, and external mutation.
- Extent and overlap of global storage.
- Whether small absolute addresses are valid mappings or reflect limitations of the static representation.
- Whether dependencies preserve the state required for subsequent accesses.
- Reachability and intended meaning of the zero-target call.
- Any guaranteed successful result, ownership outcome, or domain-level postcondition.

## Evidence

This contract is based solely on the supplied static representation. No implementation access, instruction inspection, runtime execution, or tests were performed.

The six calculated-address word writes are individually accounted for in the behavior trace and side-effect inventory. Byte writes, the halfword write, and the two G-control writes are classified separately.

Dependency IDs are local opaque occurrence labels, not verified external identities. No snapshot hash or trusted provenance value is asserted.
